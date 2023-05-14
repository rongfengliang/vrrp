# vrrp-go (fork from https://github.com/napw/VRRP-go)

由golang实现的[VRRP-v3](https://tools.ietf.org/html/rfc5798), 点击超链接获取关于VRRP的信息。
[VRRP-v3](https://tools.ietf.org/html/rfc5798) implemented by golang，click hyperlink get details about VRRP

## example

> test.go

```code
package main

import (
	"flag"
	"fmt"
	"net"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/rongfengliang/vrrp/vrrp"
	"github.com/vishvananda/netlink"
)

var (
	VRID     int
	Priority int
)

func init() {
	flag.IntVar(&VRID, "vrid", 233, "virtual router ID")
	flag.IntVar(&Priority, "pri", 100, "router priority")
}

func main() {
	flag.Parse()
	sigs := make(chan os.Signal, 1)
	done := make(chan bool, 1)

	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	var vr = vrrp.NewVirtualRouter(byte(VRID), "enp0s1", false, vrrp.IPv4)
	vr.SetPriorityAndMasterAdvInterval(byte(Priority), time.Millisecond*800)
	vr.AddIPvXAddr(net.IPv4(10, 10, 17, 29))
	vr.Enroll(vrrp.Backup2Master, func() {
		// vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrAdd(eth, addr)
		fmt.Println("backup to master")
	})
	vr.Enroll(vrrp.Init2Master, func() {
		// vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrAdd(eth, addr)
		fmt.Println("init to master")
	})
	vr.Enroll(vrrp.Master2Init, func() {
		// remove vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrDel(eth, addr)
		fmt.Println("master to init")
	})
	vr.Enroll(vrrp.Master2Backup, func() {
		// remove vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrDel(eth, addr)
		fmt.Println("master to backup")
	})
	vr.Enroll(vrrp.Init2Backup, func() {
		// remove vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrDel(eth, addr)
		fmt.Println("init  to backup")
	})
	vr.Enroll(vrrp.Backup2Init, func() {
		// remove vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrDel(eth, addr)
		fmt.Println("backup to init")
	})
	go func() {
		// starting vrrp server
		vr.StartWithEventLoop()
	}()
	go func() {
		sig := <-sigs
		fmt.Println()
		fmt.Println(sig)
		done <- true
		vr.Stop()
		// remove vip bind
		eth, _ := netlink.LinkByName("enp0s1")
		addr, _ := netlink.ParseAddr("10.10.17.29/32")
		netlink.AddrDel(eth, addr)
	}()
	fmt.Println("awaiting signal")
	<-done
	fmt.Println("exiting")
}
```