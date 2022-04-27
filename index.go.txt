package config

import (
	"log"
	"sync"

	"github.com/spf13/viper"
)

var once sync.Once

// Viper viper global instance
var Viper *viper.Viper

func init() {
	once.Do(func() {
		FlashConfig()
	})
}

// FlashConfig flash config
func FlashConfig() {
	Viper = viper.New()
	// scan the file named config in the root directory
	Viper.AddConfigPath("./")
	Viper.SetConfigName("config")

	// read config, if failed, configure by default
	if err := Viper.ReadInConfig(); err == nil {
		log.Println("Read config successfully: ", Viper.ConfigFileUsed())
	} else {
		log.Printf("Read failed: %s \n", err)
		panic(err)
	}
}

// GetUserConfig ..
func GetUserConfig(username string) (*viper.Viper, error) {
	userViper := viper.New()
	// scan the file named config in the root directory
	userViper.AddConfigPath("./user")

	userViper.SetConfigName(username)

	// read config, if failed, configure by default
	if err := userViper.ReadInConfig(); err == nil {
		log.Println("Read user config successfully: ", userViper.ConfigFileUsed())
	} else {
		log.Printf("Read user config failed: %s \n", err)
		return nil, err
	}

	return userViper, nil
}
