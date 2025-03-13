# Manipular-AS-Path-para-Influenciar-Roteamento-no-BGP

# Manipular AS-Path para Influenciar Roteamento no BGP

## **Descrição**
Projeto para manipular o peso de rotas em uma ambiente com dois links com somente  uma operadora, porém diversos AS. As configurações foram feitas em protocolos BGP.

## **Topologia**

Segue topologia utilizada. 
![topologia](Pasted%20image%2020250311093846.png)

A **REDE** cloud segue sendo um **linux Vrin** configurado para que ele possa gerar rotas ao ambiente.

![[Pasted image 20250311094038.png]]

## **Endereçamento**

Endereçamento ipv4 utilizado.

| DISPOSITIVO | INTERFACE | ENDEREÇO IPV4  |
| ----------- | --------- | -------------- |
| LOCAL       | F0/0      | 172.16.1.1/30  |
| LOCAL       | F0/1      | 172.16.1.2/30  |
| LOCAL       | Loopback0 | 9.9.9.9/32     |
| LOCAL       | Loopback1 | 8.8.8.8/32     |
| ISP5        | F0/0      | 172.16.1.2/30  |
| ISP5        | F0/1      | 10.1.2.1/30    |
| ISP5        | F1/0      | 192.168.1.1/30 |
| ISP6        | F0/0      | 10.1.2.5/30    |
| ISP6        | F0/1      | 172.16.2.2/30  |
| ISP6        | F1/0      | 192.168.1.2/30 |
| ISP7        | F0/0      | 10.1.2.6/30    |
| ISP7        | F0/1      | 10.1.2.2/30    |
| ISP7        | F1/0      | 10.1.1.2/30    |

## **Configuração**

Foram configurados de forma básica o BGP em cada roteador com a divulgação de algumas rotas, principalmente as externas da cloud **REDE**, exceto para o roteador LOCAL onde forma configurados filtros prepend e também para preferência de link com **local preference**.

### ISP5

```julia
conf t
hostname ISP5
interface FastEthernet 0/1
description IPS5 to ISP7
ip address 10.1.2.1 255.255.255.252
no shut
exit
router bgp 65005
neighbor 10.1.2.2 remote-as 65007
exit

interface FastEthernet 1/0
description IPS5 to ISP6
ip address 192.168.1.1 255.255.255.252
no shut
exit
router bgp 65005
neighbor 192.168.1.2 remote-as 65006
end

conf t
interface FastEthernet 0/0
description IPS5 to LOCAL
ip address 172.16.1.2 255.255.255.252
no shut
exit
router bgp 65005
neighbor 172.16.1.1 remote-as 65001
end

```


### ISP6

```julia
conf t
hostname ISP6
interface FastEthernet 0/0
description IPS6 to ISP7
ip address 10.1.2.5 255.255.255.252
no shut
exit
router bgp 65006
neighbor 10.1.2.6 remote-as 65007
end

conf t
interface FastEthernet 1/0
description IPS6 to ISP5
ip address 192.168.1.2 255.255.255.252
no shut
exit
router bgp 65006
neighbor 192.168.1.1 remote-as 65005
end


conf t
interface FastEthernet 0/1
description IPS6 to LOCAL
ip address 172.16.2.2 255.255.255.252
no shut
exit
router bgp 65006
neighbor 172.16.2.1 remote-as 65001
end

```

### ISP7

```julia
conf t
hostname ISP7
interface FastEthernet 1/0
description IPS7 to REDE
ip address 10.1.1.2 255.255.255.252
no shut
exit
router bgp 65007
neighbor 10.1.1.1 remote-as 65000
end

conf t
interface FastEthernet 0/1
description IPS7 to ISP5
ip address 10.1.2.2 255.255.255.252
no shut
exit
router bgp 65007
neighbor 10.1.2.1 remote-as 65005
network 10.1.2.0 mask 255.255.255.252
end

conf t
interface FastEthernet 0/0
description IPS7 to ISP6
ip address 10.1.2.6 255.255.255.252
no shut
exit
router bgp 65007
neighbor 10.1.2.5 remote-as 65006
network 10.1.2.4 mask 255.255.255.252
end

```


### LOCAL

```julia
conf t
hostname LOCAL
interface loopback 1
description Rede local
ip address 8.8.8.8 255.255.255.255
exit
ip prefix-list REDE-LOCAL seq 10 permit 8.8.8.8/32
route-map FILTRO-BGP-IN-PRI permit 10
set local-preference 350
exit
route-map FILTRO-BGP-OUT permit 10
match ip address prefix-list REDE-LOCAL
set as-path prepend 65001 65001
end


conf t
interface FastEthernet 0/0
description LOCAL to ISP5
ip address 172.16.1.1 255.255.255.252
no shut
exit
interface FastEthernet 0/1
description LOCAL to ISP6
ip address 172.16.2.1 255.255.255.252
no shut
exit
router bgp 65001
network 8.8.8.8 mask 255.255.255.255
neighbor 172.16.1.2 remote-as 65005
neighbor 172.16.1.2 route-map FILTRO-BGP-IN-PRI in
neighbor 172.16.2.2 remote-as 65006
neighbor 172.16.2.2 route-map FILTRO-BGP-OUT out
end

conf t
interface loopback 0
ip address 9.9.9.9 255.255.255.255
no shut
exit
router bgp 65001
network 9.9.9.9 mask 255.255.255.255
exit
ip prefix-list REDE-LOCAL seq 20 permit 9.9.9.9/32
end

```

## **Testes e Validação**

Comando e saídas esperados:

### ISP5

```julia
ISP5#sh running-config | se bgp
router bgp 65005
 no synchronization
 bgp log-neighbor-changes
 network 192.168.1.0 mask 255.255.255.252
 neighbor 10.1.2.2 remote-as 65007
 neighbor 172.16.1.1 remote-as 65001
 neighbor 192.168.1.2 remote-as 65006
 no auto-summary

ISP5#sh ip bgp 
BGP table version is 18, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*  2.2.2.2/32       192.168.1.2                            0 65006 65007 65000 i
*>                  10.1.2.2                               0 65007 65000 i
*> 8.8.8.8/32       172.16.1.1               0             0 65001 i
*> 9.9.9.9/32       172.16.1.1               0             0 65001 i
r  10.1.2.0/30      192.168.1.2                            0 65006 65007 i
r>                  10.1.2.2                 0             0 65007 i
*  10.1.2.4/30      192.168.1.2                            0 65006 65007 i
*>                  10.1.2.2                 0             0 65007 i
*  99.0.0.0/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.1/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.2/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.3/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.4/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.5/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.6/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.7/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.8/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*  99.0.0.9/32      192.168.1.2                            0 65006 65007 65000 ?
*>                  10.1.2.2                               0 65007 65000 ?
*> 192.168.1.0/30   0.0.0.0                  0         32768 i

```
<br/>

>Rotas como incomplete `?` são rotas geradas de forma externa, neste caso pela cloud **REDE**
<br/>

### ISP6

```julia
ISP6#sh ip bgp 
BGP table version is 20, local router ID is 192.168.1.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*  2.2.2.2/32       192.168.1.1                            0 65005 65007 65000 i
*>                  10.1.2.6                               0 65007 65000 i
*  8.8.8.8/32       10.1.2.6                               0 65007 65005 65001 i
*>                  192.168.1.1                            0 65005 65001 i
*                   172.16.2.1               0             0 65001 65001 65001 i
*  9.9.9.9/32       172.16.2.1               0             0 65001 65001 65001 i
*                   10.1.2.6                               0 65007 65005 65001 i
*>                  192.168.1.1                            0 65005 65001 i
*  10.1.2.0/30      192.168.1.1                            0 65005 65007 i
*>                  10.1.2.6                 0             0 65007 i
r  10.1.2.4/30      192.168.1.1                            0 65005 65007 i
r>                  10.1.2.6                 0             0 65007 i
*  99.0.0.0/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.1/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.2/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.3/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.4/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.5/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.6/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.7/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.8/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
*  99.0.0.9/32      192.168.1.1                            0 65005 65007 65000 ?
*>                  10.1.2.6                               0 65007 65000 ?
r  192.168.1.0/30   10.1.2.6                               0 65007 65005 i
r>                  192.168.1.1              0             0 65005 i

ISP6#sh running-config | se bgp
router bgp 65006
 no synchronization
 bgp log-neighbor-changes
 neighbor 10.1.2.6 remote-as 65007
 neighbor 172.16.2.1 remote-as 65001
 neighbor 192.168.1.1 remote-as 65005
 no auto-summary

```

### ISP7

```julia
ISP7#sh running-config | se bgp
router bgp 65007
 no synchronization
 bgp log-neighbor-changes
 network 10.1.2.0 mask 255.255.255.252
 network 10.1.2.4 mask 255.255.255.252
 neighbor 10.1.1.1 remote-as 65000
 neighbor 10.1.2.1 remote-as 65005
 neighbor 10.1.2.5 remote-as 65006
 no auto-summary

ISP7#sh ip bgp | begin Network
   Network          Next Hop            Metric LocPrf Weight Path
*> 2.2.2.2/32       10.1.1.1                 0             0 65000 i
*  8.8.8.8/32       10.1.2.5                               0 65006 65005 65001 i
*>                  10.1.2.1                               0 65005 65001 i
*  9.9.9.9/32       10.1.2.5                               0 65006 65005 65001 i
*>                  10.1.2.1                               0 65005 65001 i
*> 10.1.2.0/30      0.0.0.0                  0         32768 i
*> 10.1.2.4/30      0.0.0.0                  0         32768 i
*> 99.0.0.0/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.1/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.2/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.3/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.4/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.5/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.6/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.7/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.8/32      10.1.1.1                 0             0 65000 ?
*> 99.0.0.9/32      10.1.1.1                 0             0 65000 ?
*  192.168.1.0/30   10.1.2.5                               0 65006 65005 i
*>                  10.1.2.1                 0             0 65005 i

```

### LOCAL

```julia
LOCAL#sh ip int br
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            172.16.1.1      YES NVRAM  up                    up      
FastEthernet0/1            172.16.2.1      YES NVRAM  up                    up      
FastEthernet1/0            unassigned      YES NVRAM  administratively down down    
Loopback0                  9.9.9.9         YES manual up                    up      
Loopback1                  8.8.8.8         YES NVRAM  up                    up      

LOCAL#sh route-map 
route-map FILTRO-BGP-OUT, permit, sequence 10
  Match clauses:
    ip address prefix-lists: REDE-LOCAL 
  Set clauses:
    as-path prepend 65001 65001
  Policy routing matches: 0 packets, 0 bytes
route-map FILTRO-BGP-IN-PRI, permit, sequence 10
  Match clauses:
  Set clauses:
    local-preference 350
  Policy routing matches: 0 packets, 0 bytes

LOCAL#sh ip prefix-list 
ip prefix-list REDE-LOCAL: 2 entries
   seq 10 permit 8.8.8.8/32
   seq 20 permit 9.9.9.9/32


LOCAL#sh ip bgp | begin Network
   Network          Next Hop            Metric LocPrf Weight Path
*  2.2.2.2/32       172.16.2.2                             0 65006 65007 65000 i
*>                  172.16.1.2                    350      0 65005 65007 65000 i
*> 8.8.8.8/32       0.0.0.0                  0         32768 i
*> 9.9.9.9/32       0.0.0.0                  0         32768 i
*  10.1.2.0/30      172.16.2.2                             0 65006 65007 i
*>                  172.16.1.2                    350      0 65005 65007 i
*  10.1.2.4/30      172.16.2.2                             0 65006 65007 i
*>                  172.16.1.2                    350      0 65005 65007 i
*  99.0.0.0/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.1/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.2/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.3/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.4/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.5/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.6/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
   Network          Next Hop            Metric LocPrf Weight Path
*  99.0.0.7/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.8/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  99.0.0.9/32      172.16.2.2                             0 65006 65007 65000 ?
*>                  172.16.1.2                    350      0 65005 65007 65000 ?
*  192.168.1.0/30   172.16.2.2                             0 65006 65005 i
*>                  172.16.1.2               0    350      0 65005 i


LOCAL#sh ip bgp summary | begin Neighbor
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.1.2      4 65005      89      85       17    0    0 01:20:20       14
172.16.2.2      4 65006      90      85       17    0    0 01:20:16       14

LOCAL# sh running-config | se bgp
router bgp 65001
 no synchronization
 bgp log-neighbor-changes
 network 8.8.8.8 mask 255.255.255.255
 network 9.9.9.9 mask 255.255.255.255
 neighbor 172.16.1.2 remote-as 65005
 neighbor 172.16.1.2 route-map FILTRO-BGP-IN-PRI in
 neighbor 172.16.2.2 remote-as 65006
 neighbor 172.16.2.2 route-map FILTRO-BGP-OUT out
 no auto-summary

```

