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

## Cenário 4

Foi criado um filtro na redistribuição do **BGP** para o **OSPF** onde a **rota default** compartilhada entre os BGPs é filtrada no OSPF, sendo bloqueada.

### Configuração

#### R1

```julia
conf t
ip route 0.0.0.0 0.0.0.0 10.1.1.2
router bgp 65001
network 0.0.0.0 mask 0.0.0.0
end 
```

#### R2

```julia
conf t
ip prefix-list DEFULT-ROTA seq 10 deny 0.0.0.0/0
ip prefix-list DEFULT-ROTA seq 20 permit 0.0.0.0/1 le 32
route-map FILTRO-METRIC permit 20
no match ip address prefix-list LIBERA-TODAS
match ip address prefix-list DEFULT-ROTA
end
```

### Testes e validação

#### R1

```julia
R1#sho run | section bgp
router bgp 65001
 no synchronization
 bgp log-neighbor-changes
 network 0.0.0.0
 network 10.1.1.0 mask 255.255.255.0
 neighbor 10.1.1.2 remote-as 65000
 neighbor 192.168.1.2 remote-as 65002
 no auto-summary

R1#show ip route 
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is 10.1.1.2 to network 0.0.0.0

     2.0.0.0/32 is subnetted, 1 subnets
B       2.2.2.2 [20/0] via 10.1.1.2, 00:29:26
     172.16.0.0/32 is subnetted, 15 subnets
B       172.16.1.13 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.12 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.14 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.9 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.8 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.11 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.10 [20/0] via 10.1.1.2, 00:29:26
B       172.16.1.5 [20/0] via 10.1.1.2, 00:29:27
B       172.16.1.4 [20/0] via 10.1.1.2, 00:29:27
B       172.16.1.7 [20/0] via 10.1.1.2, 00:29:27
B       172.16.1.6 [20/0] via 10.1.1.2, 00:29:28
B       172.16.1.1 [20/0] via 10.1.1.2, 00:29:28
B       172.16.1.0 [20/0] via 10.1.1.2, 00:29:28
B       172.16.1.3 [20/0] via 10.1.1.2, 00:29:28
B       172.16.1.2 [20/0] via 10.1.1.2, 00:29:28
     10.0.0.0/24 is subnetted, 1 subnets
C       10.1.1.0 is directly connected, FastEthernet0/0
C    192.168.1.0/24 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 10.1.1.2

```

#### R2

```julia
R2#show ip bgp | include 0.0.0.0
*> 0.0.0.0          192.168.1.1              0             0 65001 i

R2#show run | section ospf
 ip ospf 2 area 0
router ospf 2
 router-id 2.2.2.2
 log-adjacency-changes
 redistribute bgp 65002 subnets route-map FILTRO-METRIC

R2#show route-map FILTRO-METRIC 
route-map FILTRO-METRIC, permit, sequence 10
  Match clauses:
    ip address (access-lists): 10 
  Set clauses:
    metric 200
  Policy routing matches: 0 packets, 0 bytes
route-map FILTRO-METRIC, permit, sequence 20
  Match clauses:
    ip address prefix-lists: DEFULT-ROTA 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes


R2#show ip prefix-list DEFULT-ROTA
ip prefix-list DEFULT-ROTA: 2 entries
   seq 10 deny 0.0.0.0/0
   seq 20 permit 0.0.0.0/1 le 32

```


#### R4

```julia
R4#show ip route ospf 
     2.0.0.0/32 is subnetted, 1 subnets
O E2    2.2.2.2 [110/1] via 10.2.2.2, 00:18:54, FastEthernet0/0
     172.16.0.0/32 is subnetted, 15 subnets
O E2    172.16.1.13 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.12 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.14 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.9 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.8 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.11 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.10 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.5 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.4 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.7 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.6 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.1 [110/200] via 10.2.2.2, 00:32:18, FastEthernet0/0
O E2    172.16.1.0 [110/200] via 10.2.2.2, 00:32:19, FastEthernet0/0
O E2    172.16.1.3 [110/200] via 10.2.2.2, 00:32:19, FastEthernet0/0
O E2    172.16.1.2 [110/200] via 10.2.2.2, 00:32:19, FastEthernet0/0
     10.0.0.0/24 is subnetted, 2 subnets
O E2    10.1.1.0 [110/1] via 10.2.2.2, 00:18:55, FastEthernet0/0

```


## Cenário 5

Foi criado uma configuração para que as rotas aprendidas pelo `R1` tivessem a sua **local preference** alterada para 200, onde antes estava como default, valor 100.

### Configuração

#### R1

```julia
conf t
ip prefix-list ROTAS-172 seq 10 permit 172.16.1.0/32
ip prefix-list ROTAS-172 seq 20 permit 172.16.1.0/23 ge 24
route-map permit 10
match ip address prefix-list ROTAS-172
set local-preference 200
exit
router bgp 65001
neighbor 10.1.1.2 route-map ALTER-PREF in
end
```


### Testes e validação
#### R1

```julia

R1#show ip bgp 172.16.1.13
BGP routing table entry for 172.16.1.13/32, version 23
Paths: (1 available, best #1, table Default-IP-Routing-Table)
Flag: 0x800
  Advertised to update-groups:
        1
  65000
    10.1.1.2 from 10.1.1.2 (2.2.2.2)
      Origin incomplete, metric 0, localpref 200, valid, external, best

R1#show ip bgp 172.16.1.14
BGP routing table entry for 172.16.1.14/32, version 22
Paths: (1 available, best #1, table Default-IP-Routing-Table)
Flag: 0x800
  Advertised to update-groups:
        1
  65000
    10.1.1.2 from 10.1.1.2 (2.2.2.2)
      Origin incomplete, metric 0, localpref 200, valid, external, best


R1#show ip bgp 
BGP table version is 35, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 0.0.0.0          10.1.1.2                 0         32768 i
*> 10.1.1.0/24      0.0.0.0                  0         32768 i
*> 172.16.1.0/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.1/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.2/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.3/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.4/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.5/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.6/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.7/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.8/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.9/32    10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.10/32   10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.11/32   10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.12/32   10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.13/32   10.1.1.2                 0    200      0 65000 ?
*> 172.16.1.14/32   10.1.1.2                 0    200      0 65000 ?

```

## Cenário 6
Aqui decidimos por restaurar o filtro na redistribuição das rotas apendidas pelo OSPF do BGP, para isso cofiguramos um filtro no comando `redistribute bgp` do OSPF no router `R2`.

### Configuração

#### R2
```julia
conf t
no route-map FILTRO-METRIC permit sequence 10
route-map FILTRO-METRIC deny 10
match ip address 10
end
```

Como já tínhamos uma `rote-map` configurada para alterar as métricas a removemos e criamos uma nova, mas utilizando a mesma ACL, porém no papel de bloqueio.

### Testes e Validação

```julia
R2#sh ip route 
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route

Gateway of last resort is 192.168.1.1 to network 0.0.0.0

     172.16.0.0/32 is subnetted, 15 subnets
B       172.16.1.13 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.12 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.14 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.9 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.8 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.11 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.10 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.5 [20/0] via 192.168.1.1, 00:48:18
B       172.16.1.4 [20/0] via 192.168.1.1, 00:48:19
B       172.16.1.7 [20/0] via 192.168.1.1, 00:48:19
B       172.16.1.6 [20/0] via 192.168.1.1, 00:48:19
B       172.16.1.1 [20/0] via 192.168.1.1, 00:48:19
B       172.16.1.0 [20/0] via 192.168.1.1, 00:48:20
B       172.16.1.3 [20/0] via 192.168.1.1, 00:48:20
B       172.16.1.2 [20/0] via 192.168.1.1, 00:48:20
     10.0.0.0/24 is subnetted, 2 subnets
C       10.2.2.0 is directly connected, FastEthernet0/0
B       10.1.1.0 [20/0] via 192.168.1.1, 00:48:20
C    192.168.1.0/24 is directly connected, FastEthernet0/1
B*   0.0.0.0/0 [20/0] via 192.168.1.1, 00:48:20
R2#
R2#sh rou
R2#sh route-map FILTRO-METRIC
route-map FILTRO-METRIC, deny, sequence 10
  Match clauses:
    ip address (access-lists): 10 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
route-map FILTRO-METRIC, permit, sequence 20
  Match clauses:
    ip address prefix-lists: DEFULT-ROTA 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
R2#
R2#
R2#sh acc
R2#sh ip acces
R2#sh ip access-lists 10
Standard IP access list 10
    10 permit 172.16.1.0, wildcard bits 0.0.0.255 (30 matches)

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

     10.0.0.0/24 is subnetted, 2 subnets
C       10.2.2.0 is directly connected, FastEthernet0/0
O E2    10.1.1.0 [110/1] via 10.2.2.2, 00:28:50, FastEthernet0/0
R4#
R4#
R4#sh ip osp
R4#sh ip ospf data
R4#sh ip ospf database 

            OSPF Router with ID (4.4.4.4) (Process ID 2)

		Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
2.2.2.2         2.2.2.2         1156        0x80000004 0x0025D0 1
4.4.4.4         4.4.4.4         1200        0x80000004 0x007870 1

		Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.2.2.1        4.4.4.4         1200        0x80000002 0x0057AA

		Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.1.1.0        2.2.2.2         1741        0x80000003 0x009F16 65001

```

