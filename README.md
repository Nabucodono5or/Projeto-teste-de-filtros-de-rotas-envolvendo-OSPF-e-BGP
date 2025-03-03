# Projeto-teste-de-filtros-de-rotas-envolvendo-OSPF-e-BGP

## **Descrição**
Projeto que visa testar diversos cenários de rotas distribuídas entre roteadores com os protocolos OPSF e BGP configurados usando de filtros com route-map.

## **Topologia**

A topologia é a mesma para todos os cenários.
![topologia](Pasted%20image%2020250224112044.png)


A **LAN** cloud segue sendo um **linux Vrin** configurado para que ele possa gerar rotas ao ambiente.

![topologia](Pasted%20image%2020250224113245.png)

configurações dos protocolos em cada roteador.

### R1

```ruby
R1#sho running-config | section ospf
R1#sho running-config | section bgp 
router bgp 65001
 no synchronization
 bgp log-neighbor-changes
 network 10.1.1.0 mask 255.255.255.0
 neighbor 10.1.1.2 remote-as 65000
 neighbor 192.168.1.2 remote-as 65002
 no auto-summary

```

### R2
```ruby

R2#sh running-config | section bgp
 redistribute bgp 65002 metric 100 subnets
router bgp 65002
 no synchronization
 bgp log-neighbor-changes
 neighbor 192.168.1.1 remote-as 65001
 no auto-summary
R2#sh running-config | section ospf
 ip ospf 2 area 0
router ospf 2
 router-id 2.2.2.2
 log-adjacency-changes
 redistribute bgp 65002 metric 100 subnets

```


### R4
```ruby
R4#sh running-config | section ospf
 ip ospf 2 area 0
router ospf 2
 router-id 4.4.4.4
 log-adjacency-changes

R4#sh running-config interface fastEthernet 0/0
Building configuration...

Current configuration : 130 bytes
!
interface FastEthernet0/0
 description to R2
 ip address 10.2.2.1 255.255.255.0
 ip ospf 2 area 0
 duplex auto
 speed auto
end
```

## **Endereçamento**

Endereçamento ipv4  utilizados nos cenários

| DISPOSITIVO | INTERFACE | ENDEREÇO IPV4  |
| ----------- | --------- | -------------- |
| R1          | F0/0      | 10.1.1.1/24    |
| R1          | F0/1      | 192.168.1.1/24 |
| R2          | F0/0      | 10.2.2.2/24    |
| R2          | F0/1      | 192.168.1.2/24 |
| R4          | F0/0      | 10.2.2.1/24    |
| LAN         | E0        | 10.1.1.2/24    |
## Cenário 1

Filtro para todo endereço, exceto o ip `172.16.1.14/32` aprendido da LAN no roteador *R1* para o peer `192.168.1.2 `

### Configuração
#### R1

```julia
conf t
ip prefix-list FILTRO-BGP-EX seq 10 deny 172.16.1.14/32
ip prefix-list FILTRO-BGP-EX seq 20 permit 0.0.0.0 le 32
route-map FILTRO-BGP-OUT deny 10
match ip address prefix-list FILTRO-BGP-EX
router bgp 65001
neighbor 192.168.1.2 route-map FILTRO-BGP-OUT out
end
```

### Testes e validação

Comando e saídas esperadas:
#### R1

```julia
R1#sho ip bgp neighbors 192.168.1.2 advertised-routes 
BGP table version is 18, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 172.16.1.14/32   10.1.1.2                 0             0 65000 ?

Total number of prefixes 1
```

#### R2

```julia
R2#show ip route 
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/32 is subnetted, 15 subnets
B       172.16.1.14 [20/0] via 192.168.1.1, 01:26:12
     10.0.0.0/24 is subnetted, 2 subnets
C       10.2.2.0 is directly connected, FastEthernet0/0
B       10.1.1.0 [20/0] via 192.168.1.1, 01:16:42
C    192.168.1.0/24 is directly connected, FastEthernet0/1
```

## Cenário 2

Filtrando no R2 as rotas aprendidas do vizinho R1 `192.168.1.1`, o R1 continua informando as rotas, porém o R2 filtra o que está sendo recebido e aloca somente o IP `172.16.1.14` no **RIB**.

### Configuração

#### R1

```julia
conf t
router bgp 65001
no neighbor 192.168.1.2 route-map FILTRO-REDES-BGP out
end

clear ip bgp 192.168.1.2 soft out
```

#### R2

```julia
conf t
ip prefix-list FILTRO-BGP-IN seq 10 permit 172.16.1.14/32
ip prefix-list FILTRO-BGP-IN seq 20 deny 0.0.0.0/0 le 32
route-map FILTRO-BGP-REDES permit 10
match ip address prefix-list FILTRO-BGP-IN
exit
router bgp 65002
neighbor 192.168.1.1 route-map FILTRO-BGP-REDES in
end

sh ip bgp neighbor 192.168.1.1 routes
sh ip route

```


### Testes e validação
Comando e saídas esperadas:
#### R1

```julia
R1#sh ip bgp neighbors 192.168.1.2 advertised-routes 
BGP table version is 18, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 2.2.2.2/32       10.1.1.2                 0             0 65000 i
*> 10.1.1.0/24      0.0.0.0                  0         32768 i
*> 172.16.1.0/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.1/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.2/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.3/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.4/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.5/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.6/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.7/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.8/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.9/32    10.1.1.2                 0             0 65000 ?
*> 172.16.1.10/32   10.1.1.2                 0             0 65000 ?
*> 172.16.1.11/32   10.1.1.2                 0             0 65000 ?
*> 172.16.1.12/32   10.1.1.2                 0             0 65000 ?
*> 172.16.1.13/32   10.1.1.2                 0             0 65000 ?
*> 172.16.1.14/32   10.1.1.2                 0             0 65000 ?
   Network          Next Hop            Metric LocPrf Weight Path

Total number of prefixes 17 

```

#### R2

```julia
R2#sh route-map 
route-map FILTRO-BGP-REDES, permit, sequence 10
  Match clauses:
    ip address prefix-lists: FILTRO-BGP-IN 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
R2#sh ip route       
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/32 is subnetted, 1 subnets
B       172.16.1.14 [20/0] via 192.168.1.1, 02:02:33
     10.0.0.0/24 is subnetted, 1 subnets
C       10.2.2.0 is directly connected, FastEthernet0/0
C    192.168.1.0/24 is directly connected, FastEthernet0/1


R2#sh ip bgp neighbors 192.168.1.1 routes 
BGP table version is 34, local router ID is 192.168.1.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 172.16.1.14/32   192.168.1.1                            0 65001 65000 ?

Total number of prefixes 1 

```

#### R4

```julia
R4#sh ip route 
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     172.16.0.0/32 is subnetted, 1 subnets
O E2    172.16.1.14 [110/100] via 10.2.2.2, 02:03:03, FastEthernet0/0
     10.0.0.0/24 is subnetted, 1 subnets
C       10.2.2.0 is directly connected, FastEthernet0/0

```

## Cenário 3

No roteador `R2`, estamos redistribuindo rotas aprendidas pelo **BGP** para **OSPF**, mas queremos aumentar a métrica delas para **200** através de **route-map**.

### configuração da redistribuição (BGP -> OSPF) no R2
#### R2

```julia
R2#show running-config | section ospf
 ip ospf 2 area 0
router ospf 2
 router-id 2.2.2.2
 log-adjacency-changes
 redistribute bgp 65002 metric 100 subnets

```


### Configuração
#### R2

```julia
conf t
router bgp 65002
no neighbor 192.168.1.1 route-map FILTRO-BGP-REDES in
end


conf t
access-list 10 permit 172.16.1.0 0.0.0.255
route-map FILTRO-METRIC permit 10
match ip address 10 
set metric 200
exit
router ospf 2
no redistribute bgp 65002 metric 100 subnets
redistribute bgp 65002 subnets route-map FILTRO-METRIC
end

conf t
ip prefix-list LIBERA-TODAS seq 10 permit 0.0.0.0/0 le 32
route-map FILTRO-METRIC permit 20
match ip address prefix-list LIBERA-TODAS
end

```

### Testes e validação

#### R4

```julia
R4#show ip route 
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is not set

     2.0.0.0/32 is subnetted, 1 subnets
O E2    2.2.2.2 [110/1] via 10.2.2.2, 00:01:06, FastEthernet0/0
     172.16.0.0/32 is subnetted, 15 subnets
O E2    172.16.1.13 [110/200] via 10.2.2.2, 02:02:51, FastEthernet0/0
O E2    172.16.1.12 [110/200] via 10.2.2.2, 02:02:51, FastEthernet0/0
O E2    172.16.1.14 [110/200] via 10.2.2.2, 02:02:51, FastEthernet0/0
O E2    172.16.1.9 [110/200] via 10.2.2.2, 02:02:51, FastEthernet0/0
O E2    172.16.1.8 [110/200] via 10.2.2.2, 02:02:51, FastEthernet0/0
O E2    172.16.1.11 [110/200] via 10.2.2.2, 02:02:52, FastEthernet0/0
O E2    172.16.1.10 [110/200] via 10.2.2.2, 02:02:52, FastEthernet0/0
O E2    172.16.1.5 [110/200] via 10.2.2.2, 02:02:52, FastEthernet0/0
O E2    172.16.1.4 [110/200] via 10.2.2.2, 02:02:52, FastEthernet0/0
O E2    172.16.1.7 [110/200] via 10.2.2.2, 02:02:52, FastEthernet0/0
O E2    172.16.1.6 [110/200] via 10.2.2.2, 02:02:54, FastEthernet0/0
O E2    172.16.1.1 [110/200] via 10.2.2.2, 02:02:54, FastEthernet0/0
O E2    172.16.1.0 [110/200] via 10.2.2.2, 02:02:54, FastEthernet0/0
O E2    172.16.1.3 [110/200] via 10.2.2.2, 02:02:54, FastEthernet0/0
O E2    172.16.1.2 [110/200] via 10.2.2.2, 02:02:54, FastEthernet0/0
     10.0.0.0/24 is subnetted, 2 subnets
C       10.2.2.0 is directly connected, FastEthernet0/0
O E2    10.1.1.0 [110/1] via 10.2.2.2, 00:01:09, FastEthernet0/0

```

* As métricas na tabela de **OSPF** foram todas alteradas para 200.
#### R2

```julia
R2#show running-config | section ospf
 ip ospf 2 area 0
router ospf 2
 router-id 2.2.2.2
 log-adjacency-changes
 redistribute bgp 65002 subnets route-map FILTRO-METRIC
R2#show access-lists                 
Standard IP access list 10
    10 permit 172.16.1.0, wildcard bits 0.0.0.255 (15 matches)
R2#show route-map FILTRO-METRIC
route-map FILTRO-METRIC, permit, sequence 10
  Match clauses:
    ip address (access-lists): 10 
  Set clauses:
    metric 200
  Policy routing matches: 0 packets, 0 bytes
route-map FILTRO-METRIC, permit, sequence 20
  Match clauses:
    ip address prefix-lists: LIBERA-TODAS 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
```

