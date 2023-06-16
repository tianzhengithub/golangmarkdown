## WSl中docker启动的服务本地无法访问

> 建立win10和wsl中的linux的端口映射关系
>上述步骤只能在wsl中访问容器的8080端口，你可以通过 ifconfig 查看docker的ip，然后通过curl docker-ip即可看到服务的返回。但是直接在win10的浏览器上访问还>是不行，因此需要建立映射。
>以管理员身份打开powershell

>netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=8077 connectaddress=<wsl中的linux的eth0的ip> connectport=<wsl中linux映射的端口>
>#例如：上面我是用wsl的ubuntu中的80端口映射到了容器的8080端口，所以我这边的connectport要填80，172.19.115.157是我ubuntu的eth0的ip
>netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=8077 connectaddress=172.19.115.157 connectport=80
