---
layout: single
title:  "Shell Script로 쿠버네티스 CNI 플러그인 만들기"
date:   2023-01-08 12:00:00 +0900
categories: 
tags: Kubernetes
---

**CNI**(Container Networking Interface)는 컨테이너의 네트워크 인터페이스를 구성하기 위한 외부 모듈이다. 쿠버네티스는 Calico, Flannel, Weave Net과 같은 CNI 플러그인으로 네트워크를 구성한다. 이들은 공통적으로 [CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)을 구현하도록 만들어졌다. 내부 구현은 달라도 인터페이스는 동일한 것이다.

그렇다면, Bash Script로 CNI 플러그인을 구현할 수 있지 않을까?

> 이 글은 유럽 KubeCon 2019의 [Kubernetes Networking: How to Write a CNI Plugin From Scratch](https://youtube.com/watch?v=zmYxdtFzK6s) 세션의 내용을 1.24 이후 버전의 쿠버네티스에 맞게 고친 것이다.

> **쿠버네티스 네트워크 모델**과 **리눅스 네임스페이스** 개념을 미리 알아두면 좋다. [이 글](https://www.44bits.io/ko/post/container-network-2-ip-command-and-network-namespace)에 설명이 잘 되어있다.


## CNI 플러그인의 작동 방식

CNI 플러그인은 컨테이너 런타임이 **CNI 바이너리를 실행**하는 방식으로 작동한다. CNI에는 런타임이 바이너리 실행 시 어떤 파라미터를 어떤 형식으로 전달해야 하는지 정의되어있다. 예를 들어 `CNI_COMMAND` 환경 변수에다가 실행할 작업(`ADD`, `DEL`, `CHECK` 및 `VERSION`)을 세팅하면 CNI 바이너리가 해당 값을 읽어 필요한 작업을 실행하는 방식이다. 

셸 스크립트로는 이런 형태가 될 것이다.

```bash
#!/bin/bash
case $CNI_COMMAND in
ADD)
    # 새로운 컨테이너에 대한 네트워크 구성
    ;;
DEL)
    # 컨테이너가 중지되었을 때 자원 제거
    ;;
VERSION)
    # 지원하는 CNI 버전 정보 출력
    ;;
esac
```

런타임은 이걸 실행만 할 뿐, 내부 구현에는 관심 없으므로 바이너리든 스크립트든 작동에는 상관이 없다.

> `CHECK` 커맨드는 이 글에서 다루지 않는다.


쿠버네티스가 Pod를 실행할 때 CNI 플러그인을 호출하는 과정을 요약하면 다음과 같다.

1. 쿠버네티스가 컨테이너 런타임에게 컨테이너 생성 요청.
1. 컨테이너 런타임이 `pause` 컨테이너와 네트워크 네임스페이스 생성.
1. 컨테이너 런타임이 환경 변수 설정. (`CNI_COMMAND=ADD` 등등)
1. 컨테이너 런타임이 CNI 플러그인 실행.
1. 컨테이너 런타임이 플러그인에게 `stdin`으로 CNI config 전달.
1. CNI 플러그인이 환경 변수와 `stdin` 입력을 읽어들임.
1. CNI 플러그인이 네트워크 인터페이스 구성.


## 구현

### 실습 환경
- 노드 2개로 구성된 쿠버네티스 클러스터 (`node1`, `node2`)
- 쿠버네티스 버전 1.24
- 클러스터의 Pod CIDR: `10.240.0.0/16`
- `containerd` 런타임
- `jq`가 설치됨

> AWS EKS 환경에서는 잘 안 된다. 이유는 모르겠는데, 뭔가 EKS 자체적인 설정이 있는 것 같다. AWS EC2나 로컬 VirtualBox 환경에서 진행하자.


### 컨테이너 런타임 설정

먼저 `containerd`가 CNI 플러그인을 써먹을 수 있도록 설정을 해줘야 한다. 아마 기본값이 잘 설정되어 있겠지만, `/etc/containerd/config.toml` 파일을 열어봐서 아래 내용이 있는지 확인해보자.

```toml
[plugins."io.containerd.grpc.v1.cri".cni]
bin_dir = "/opt/cni/bin"
conf_dir = "/etc/cni/net.d"
```


### Config

1번 노드에 `/etc/cni/net.d/10-my-cni-demo.conf` 파일을 생성한다. 내용은 아래처럼.

```json
{
    "cniVersion": "0.3.1",
    "name": "my-network",
    "type": "my-cni-demo",
    "podcidr": "10.240.0.0/24"
}
```

`type`에는 플러그인의 이름이 들어간다. `podcidr`는 사용자 정의 필드로 플러그인에서 읽어들일 수 있게 하드코딩으로 집어넣은 값이다.

2번 노드에도 같은 경로에 같은 내용의 파일을 생성하되, `podcidr`의 값을 `"10.240.1.0/24"`으로 설정한다.

> 실습할 클러스터의 Pod CIDR가 `10.240.0.0/16`이 아닌 다른 값이라면 그에 맞게 내용을 수정하자.


### 스크립트 작성

`/opt/cni/bin/` 디렉터리에 `my-cni-demo` 파일을 하나 생성한다. 이 파일이 바로 컨테이너 런타임이 사용할 CNI 플러그인이다. 편집기로 파일을 열고 스크립트를 작성해보자. 

먼저 ADD 커맨드를 처리한다. 컨테이너 네트워크가 사용할 브리지인 `cni0`를 생성한다.

```bash
#!/bin/bash
config=`cat /dev/stdin`

case $CNI_COMMAND in
ADD)
    podcidr=$(echo $config | jq -r ".podcidr")  # 10.240.0.0/24 (주석은 1번 노드 기준)
    podcidr_gw=$(echo $podcidr | sed "s:0/24:1:g")  # 10.240.0.1

    # 컨테이너 네트워크를 위한 브리지 구성
    ip link add cni0 type bridge  # cni0라는 이름의 브리지 생성
    ip addr add "${podcidr_gw}/24" dev cni0  # cni0에 10.240.0.1/24 할당
    ip link set cni0 up
```

사실 이건 딱 한 번만 만들면 된다. `cni0`가 이미 있으면 아무 작업도 하지 않을 것이다.

이제 가상 인터페이스를 만들어 Pod와 브리지를 연결하자. `$CNI_IFNAME` 인터페이스는 Pod 쪽에, `$host_ifname` 인터페이스는 호스트의 브리지 쪽에 붙인다. 아마 `eth0`-`veth1`과 같은 형태의 veth 쌍이 생성될 것이다.

```bash
    if [ -f /tmp/last_allocated_ip ]; then
        n=`cat /tmp/last_allocated_ip`
    else
        n=1
    fi
    n=$(($n+1))  # n=2,3,4,... (255를 넘어가면 고장난다)
    echo $n > /tmp/last_allocated_ip

    # veth 쌍 생성 및 활성화
    host_ifname="veth$n"  # veth2, veth3, veth4, ...
    netns=$(echo $CNI_NETNS | sed "s:/var/run/netns/::")
    ip netns exec $netns ip link add $CNI_IFNAME type veth peer name $host_ifname
    ip netns exec $netns ip link set $CNI_IFNAME up

    ip netns exec $netns ip link set $host_ifname netns 1
    ip link set $host_ifname up
    ip link set $host_ifname master cni0
```

여기서 유의할 점은 Pod의 네트워크 네임스페이스는 이미 생성되어 있다는 것이다. 쿠버네티스는 CNI 플러그인을 실행하기에 앞서 **`pause` 컨테이너**를 생성함으로써 네트워크 네임스페이스를 생성한다. 해당 네임스페이스와 인터페이스 이름이 각각 `CNI_NETNS`, `CNI_IFNAME` 환경 변수에 할당되어 플러그인이 이 값을 읽어들일 수 있다.

`CNI_NETNS`에는 `/var/run/netns/cni-6f22546b-6404-cf51-7ef5-5db22bff5f86`과 같은 값이 들어있다. 해당 경로에는 `pause` 컨테이너가 속한 네트워크 네임스페이스가 정의되어있다. 이를 `ip netns` 도구로 관리할 수 있다.

> 이 글에서는 `pause` 컨테이너에 대해 자세히 다루진 않는다. 네트워크 네임스페이스를 생성하는 역할 정도로 이해하면 된다.

이제 Pod 네임스페이스에 붙은 인터페이스에 IP와 라우트를 설정하자.

```bash
    ip=$(echo $podcidr | sed "s:0/24:$n:g")
    ip netns exec $netns ip addr add $ip/24 dev $CNI_IFNAME
    ip netns exec $netns ip route add default via $podcidr_gw
```

마지막으로 실행 결과를 출력하는 부분이다.

```bash
    mac=$(ip netns exec $netns ip link show eth0 | awk '/ether/ {print $2}')
    address="${ip}/24"
    output_template=$(cat <<END
{
    "cniVersion": "0.3.1",
    "interfaces": [
        {
            "name": "%s",
            "mac": "%s",
            "sandbox": "%s" 
        }
    ],
    "ips": [
        {
            "version": "4",
            "address": "%s",
            "gateway": "%s",
            "interface": 0
        }
    ]
}
END
)
    output=$(printf "$output_template" "$CNI_IFNAME" "$mac" "$CNI_NETNS" "$address" "$podcidr_gw")
    echo "$output" | tee -a $log
    ;;
```

이상 `ADD` 커맨드에 대한 스크립트를 작성했다.

`DEL`과 `VERSION` 커맨드도 구현해보자. 별건 없다.

```bash
DEL)
    # 네임스페이스와 그에 연결된 인터페이스 삭제
    netns=$(echo $CNI_NETNS | sed "s:/var/run/netns/::")
    ip netns delete $netns
    ;;
VERSION)
    echo '{
    "cniVersion": "0.3.1",
    "supportedVersions": ["0.3.0", "0.3.1", "0.4.0"]
}'
    ;;
*)
    echo "Unknown cni command: $CNI_COMMAND" 
    exit 1
    ;;
esac
```

이제 이 스크립트를 실행할 수 있도록 `chmod +x my-cni-demo`를 해주자.


## Pod 배포

`demo.yaml` 파일을 작성 후 `kubectl apply -f demo.yaml`로 Pod를 생성하자.

<details><summary>demo.yaml 파일 내용 보기</summary>  

{% highlight yaml %}
# demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpine1
spec:
  containers:
  - name: alpine
    image: alpine
    command:
      - "/bin/ash"
      - "-c"
      - "sleep 1000000"
  nodeSelector:
    kubernetes.io/hostname: node1  # 1번 노드
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
spec:
  containers:
  - name: nginx
    image: nginx:alpine
  nodeSelector:
    kubernetes.io/hostname: node1  # 1번 노드
---
apiVersion: v1
kind: Pod
metadata:
name: nginx2
spec:
  containers:
  - name: nginx
    image: nginx:alpine
  nodeSelector:
    kubernetes.io/hostname: node2  # 2번 노드
{% endhighlight %}

</details>  


Pod의 작동 상태를 확인한다.

```console
$ kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
alpine1   1/1     Running   0          10h   10.240.0.2   node1   <none>           <none>
nginx1    1/1     Running   0          10h   10.240.0.3   node1   <none>           <none>
nginx2    1/1     Running   0          10h   10.240.1.2   node2   <none>           <none>
```

`ping`으로 Pod 의 통신 상태를 확인해보자

```console
$ kubectl exec alpine1 -- ping 10.240.0.3 -c 3  # 같은 노드 (alpine1 <--> nginx1)
PING 10.240.0.3 (10.240.0.3): 56 data bytes
64 bytes from 10.240.0.3: seq=0 ttl=255 time=0.093 ms
64 bytes from 10.240.0.3: seq=1 ttl=255 time=0.085 ms
64 bytes from 10.240.0.3: seq=2 ttl=255 time=0.083 ms
```

```console
$ kubectl exec alpine1 -- ping 10.240.1.2 -c 3  # 다른 노드 (alpine1 <--> nginx2)
PING 10.240.1.2 (10.240.1.2): 56 data bytes
```

```console
$ kubectl exec alpine1 -- ping 1.1.1.1 -c 3  # 외부 인터넷 (alpine1 <--> 1.1.1.1)
PING 1.1.1.1 (1.1.1.1): 56 data bytes
64 bytes from 1.1.1.1: seq=0 ttl=48 time=1.323 ms
64 bytes from 1.1.1.1: seq=1 ttl=48 time=1.331 ms
64 bytes from 1.1.1.1: seq=2 ttl=48 time=1.300 ms
```

아직 다른 노드 간의 통신이 되지 않는 상황이다.

> 혹시 같은 노드 내에서의 통신이나 외부 인터넷과의 통신이 되지 않는다면 `iptables` 규칙을 건드려보자.
> ```sh
> # Allow pod to pod communication
> iptables -A FORWARD -s 10.240.0.0/16 -j ACCEPT
> iptables -A FORWARD -d 10.240.0.0/16 -j ACCEPT
> # Allow outgoing internet 
> iptables -t nat -A POSTROUTING -s 10.240.0.0/24 ! -o cni0 -j MASQUERADE
> ```


## 노드 간 라우트 

지금까지 구현한 스크립트는 **네트워크 인터페이스**를 설정하는 코드였다. 그렇지만 위에서 만든 것만 갖고는 서로 다른 노드간에 통신이 불가능하다. 1번 노드는 `10.240.1.2`로 향하는 패킷을 어디로 보내야 하는지 아는 게 없기 때문이다.

일반적인 CNI 플러그인은 각 노드마다 **데몬**이 돌면서 네트워크 세팅을 관리한다. Flannel의 `flanneld`가 대표적인 예시이다. 이 글에선 데몬까지 구현하진 않고, 각 노드마다 직접 명령을 실행하는 식으로 간단히 진행해본다.

1번 노드에서 실행. 노드 IP는 본인 환경에 맞게 바꾼다.

```console
# modprobe ipip
# ip tunnel add ipip0 mode ipip remote $NODE2_IP local $NODE1_IP
# ip addr add 10.240.0.1/24 dev ipip0
# ip link set ipip0 up
# ip route add 10.240.1.0/24 dev ipip0
```

2번 노드에서 실행.

```console
# modprobe ipip
# ip tunnel add ipip0 mode ipip remote $NODE1_IP local $NODE2_IP
# ip addr add 10.240.1.1/24 dev ipip0
# ip link set ipip0 up
# ip route add 10.240.0.0/24 dev ipip0
```

이제 노드 간 통신도 잘 될 것이다.

```console
$ kubectl exec alpine1 -- ping 10.240.1.2 -c 3
PING 10.240.1.2 (10.240.1.2): 56 data bytes
64 bytes from 10.240.1.2: seq=0 ttl=255 time=0.683 ms
64 bytes from 10.240.1.2: seq=1 ttl=255 time=0.531 ms
64 bytes from 10.240.1.2: seq=2 ttl=255 time=0.537 ms
```

여기선 **IPIP 터널링**을 활용했다. 실제 CNI 플러그인은 IPIP, VXLAN, BGP 등의 방식을 활용한다. 근데 아직 내가 공부가 부족한 관계로 설명은 패스. 다음에 기회 될 때 정리해봐야겠다.


이렇게 간단한 형태의 CNI 플러그인을 만들어봤다. 당연한 소리지만 실용적인 면에서는 아무 쓸모가 없다. 그래도 직접 명령어를 치면서 네트워크를 건드려보면 CNI를 더 깊이 이해할 수 있을 것이다.


---
### 참고 자료
<https://github.com/containernetworking/cni/blob/spec-v1.0.0/SPEC.md>  
<https://github.com/eranyanay/cni-from-scratch>  
<https://itnext.io/kubernetes-networking-behind-the-scenes-39a1ab1792bb>  
<https://www.44bits.io/ko/post/container-network-2-ip-command-and-network-namespace>  
