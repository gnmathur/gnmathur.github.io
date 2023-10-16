+++
title = "Deleting a stale container"
date = "2023-11-15"
description = "Sample article showcasing basic Markdown syntax and formatting for HTML elements."
tags = [
    "docker",
]
categories = [
    "recipes",
    "docker",
]
series = ["docker"]
draft = false
+++

Oftentimes you want to remove the stopped or exited containers that you don't intend to start again. Here's a way to do this -

```
$ docker rm `docker ps -a -q -f "status=exited"`
```

For example, 

`docker ps -a` gives you the list the following containers

```
> git:(main) docker ps -a
5eca36be7c12   87689a8e1265    "/__cacert_entrypoin…"   7 hours ago    Exited (130) 7 hours ago              competent_torvalds                                                                             
e1093949e6cb   bec30be74fb3    "/__cacert_entrypoin…"   7 hours ago    Exited (1) 7 hours ago                unruffled_wiles                                                                                
8a0f435c8a5d   b686192cfc5f    "/__cacert_entrypoin…"   8 hours ago    Exited (130) 8 hours ago              keen_bardeen                                                                                   
c047cfa3b253   saarekaam:0.1   "/__cacert_entrypoin…"   29 hours ago   Exited (137) 29 hours ago             tender_goldberg                                                                                
80bd9088b788   19b6e73abb2b    "/__cacert_entrypoin…"   29 hours ago   Exited (137) 29 hours ago             eager_albattani                                                                                
46a1d61011af   19b6e73abb2b    "/__cacert_entrypoin…"   32 hours ago   Exited (137) 31 hours ago             brave_fermat                                                                                   
0d2c1071066e   f0fc0f94729e    "/__cacert_entrypoin…"   32 hours ago   Exited (1) 32 hours ago               beautiful_sammet                                                                               
f60a7321ab32   2769272fbbc8    "/__cacert_entrypoin…"   32 hours ago   Exited (1) 32 hours ago               jovial_heisenberg                                                                              
181aff5f575b   2769272fbbc8    "/__cacert_entrypoin…"   32 hours ago   Exited (0) 32 hours ago               focused_borg                                                                                   
e13f55715c2b   2769272fbbc8    "/__cacert_entrypoin…"   32 hours ago   Exited (1) 32 hours ago               lucid_bartik                                                                                   
581ef651d8ce   9e7ae85c6825    "/__cacert_entrypoin…"   32 hours ago   Exited (0) 32 hours ago               jolly_shaw                                                                                     
a86927a05147   9e7ae85c6825    "/__cacert_entrypoin…"   32 hours ago   Exited (1) 32 hours ago               eager_carver                                                                                   
f4c3784627f1   5a7c50d65bc7    "/__cacert_entrypoin…"   32 hours ago   Exited (0) 32 hours ago               xenodochial_keldysh                                                                            
27926181cc78   5a7c50d65bc7    "/__cacert_entrypoin…"   32 hours ago   Exited (1) 32 hours ago               romantic_jennings                                                                              
301348b8c935   83bc63193f8c    "/__cacert_entrypoin…"   32 hours ago   Exited (0) 32 hours ago               competent_cori                                                                                 
f6143e6f8bc1   83bc63193f8c    "/__cacert_entrypoin…"   32 hours ago   Exited (1) 32 hours ago               funny_dijkstra                                                                                 
6b1ddcfb15af   gnms:0.1        "pypy3 main.py"          3 weeks ago    Exited (0) 3 weeks ago                gnms                                                                                           
e09de0c5d5a8   fcfdcd822485    "--name gnms"            3 weeks ago    Created                               dreamy_ramanujan                                                                               
5971e11e121b   fcfdcd822485    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                focused_khayyam                                                                                
43883ac4b298   fcfdcd822485    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                heuristic_mirzakhani                                                                           
3aa5b7c8fa31   4ecbcedae7d3    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                magical_liskov                                                                                 
cb4770e494e9   4ecbcedae7d3    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                stoic_zhukovsky                                                                                
e221d6a047e0   bbe5f44bd0ca    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                exciting_saha                                                                                  
311e875a18c8   09c069097d92    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                practical_dubinsky                                                                             
f03a192b7863   657ddfb7361a    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                peaceful_jepsen                                                                                
4d553ecee119   8a7a7c872a4b    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                interesting_lalande                                                                            
82589ec89861   8a7a7c872a4b    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                vigilant_mayer                                                                                 
08eee73971f2   09db4d71eae6    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                magical_cannon                                                                                 
83c108a3b72c   0a31c16014a1    "python3 main.py"        3 weeks ago    Exited (0) 3 weeks ago                fervent_goldstine 
```

Run the command to remove the "stale" containers

```
> git:(main) docker rm `docker ps -a -q -f "status=exited"`
5eca36be7c12
e1093949e6cb
8a0f435c8a5d
c047cfa3b253
80bd9088b788
46a1d61011af
0d2c1071066e
f60a7321ab32
181aff5f575b
e13f55715c2b
581ef651d8ce
a86927a05147
f4c3784627f1
27926181cc78
301348b8c935
f6143e6f8bc1
6b1ddcfb15af
5971e11e121b
43883ac4b298
3aa5b7c8fa31
cb4770e494e9
e221d6a047e0
311e875a18c8
f03a192b7863
4d553ecee119
82589ec89861
08eee73971f2
83c108a3b72c
```

And now, when you list the containers, you'll notice that the exited ones are cleaned up!

```
> git:(main) docker ps -a
CONTAINER ID   IMAGE          COMMAND         CREATED       STATUS    PORTS     NAMES
e09de0c5d5a8   fcfdcd822485   "--name gnms"   3 weeks ago   Created             dreamy_ramanujan
```
