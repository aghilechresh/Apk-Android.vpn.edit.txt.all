package main

import (
	"encoding/base64"
	"encoding/json"
	"flag"
	"io/ioutil"
	"log"
	"net/http"
	"net/url"
	"strconv"
	"strings"
)

var subURL string

type Config struct {
	Server        string `json:"server"`
	ServerPort    int    `json:"server_port"`
	Method        string `json:"method"`
	Protocol      string `json:"protocol"`
	ProtocolParam string `json:"protocol_param"`
	OBFS          string `json:"obfs"`
	OBFSParam     string `json:"obfs_param"`
	Password      string `json:"password"`
	Remarks       string `json:"remarks"`
	Group         string `json:"group"`

	LocalAddress string `json:"local_address"`
	LocalPort    int    `json:"local_port"`
	Timeout      int    `json:"timeout"`
	UDPTimeout   int    `json:"udp_timeout"`
}

func init() {
	flag.StringVar(&subURL, "s", "https://example.com/ssr", "subscribe url")
	flag.Parse()
}

func main() {
	resp, err := http.Get(subURL)
	if err != nil {
		log.Printf("request sub url error: %s", err)
		return
	}
	reader := base64.NewDecoder(base64.RawStdEncoding, resp.Body)
	data, err := ioutil.ReadAll(reader)
	if err != nil {
		log.Printf("request body read error: %s", err)
		return
	}
	uris := strings.Split(string(data), "\n")

	for _, uri := range uris {
		if uri == "" {
			continue
		}
		if node, err := decodeURI(uri); err != nil {
			log.Printf("decode error: %s, will skip", err)
		} else {
			var nodeName string
			i := strings.Index(node.Remarks, " ")
			if i > 0 {
				nodeName = node.Remarks[:i]
			} else {
				nodeName = node.Remarks
			}
			nodeName += ".json"
			bs, _ := json.MarshalIndent(node, "", " ")
			_ = ioutil.WriteFile(nodeName, bs, 0644)
		}
	}
	log.Println("subscribe success")
}

func decodeURI(uri string) (*Config, error) {
	head := "ssr://"
	if idx := strings.Index(uri, head); idx != -1 {
		uri = uri[idx+len(head):]
	}

	b, err := base64.RawURLEncoding.DecodeString(uri)
	if err != nil {
		return nil, err
	}

	s := string(b)
	c := &Config{
		LocalAddress: "0.0.0.0",
		LocalPort:    1080,
		Timeout:      300,
		UDPTimeout:   60,
	}

	i := strings.Index(s, ":")
	if i > -1 {
		c.Server = s[:i]
		s = s[i+1:]
	}
	i = strings.Index(s, ":")
	if i > -1 {
		c.ServerPort, _ = strconv.Atoi(s[:i])
		s = s[i+1:]
	}
	i = strings.Index(s, ":")
	if i > -1 {
		c.Protocol = s[:i]
		s = s[i+1:]
	}
	i = strings.Index(s, ":")
	if i > -1 {
		c.Method = s[:i]
		s = s[i+1:]
	}
	i = strings.Index(s, ":")
	if i > -1 {
		c.OBFS = s[:i]
		s = s[i+1:]
	}
	i = strings.Index(s, "/")
	if i > -1 {
		c.Password = base64Decode(s[:i])
		s = s[i+1:]
	}

	u, err := url.Parse(s)
	if err != nil {
		return nil, err
	}
	c.OBFSParam = base64Decode(u.Query().Get("obfsparam"))
	c.ProtocolParam = base64Decode(u.Query().Get("protoparam"))
	c.Remarks = base64Decode(u.Query().Get("remarks"))
	c.Group = base64Decode(u.Query().Get("group"))

	return c, nil
}

func base64Decode(in string) string {
	if in != "" {
		b, err := base64.RawURLEncoding.DecodeString(in)
		if err == nil {
			in = string(b)
		}
	}
	return in
}
