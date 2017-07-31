---
layout: single
title:  "Resolvendo problemas do *Docker for Mac* com containers"
date:   2017-07-31 15:00:00 -0300
---
Por muito tempo o Docker não utilizou a virtualização nativa do *macOS*, o que o *Docker Machine* fazia na realidade era criar uma máquina virtual no *VirtualBox* e executar seus containers dentro da mesma, o que nem de longe é uma solução boa e realmente estável se comparado aos containers nativos do Linux, isso trazia muitos problemas de performance e te obrigava a utilizar comandos pouco intuitivos para utilizar containers no *macOS*, mas ainda assim era uma opção que funcionava.

Com a chegada do *Docker for Mac* os containers continuam sendo virtualizados, porém de forma nativa através do *HyperKit* do *macOS*, isso melhorou muito o funcionamento do Docker no sistema e acabou de vez com o *Docker Machine*, mas isso não quer dizer que todos os problemas do Docker no *macOS* foram extintos e nas últimas semanas de trabalho me deparei um com deles.

Os sintoma que eu mais enfrentei era a lentidão para conectar o container da minha aplicação com os containers de serviços dos quais ela depende, como [Redis](https://redis.io/) e [PostgreSQL](https://www.postgresql.org/). No pior dos casos os serviços não conectavam nunca e no melhor demorava mais de 10 minutos para que o meu ambiente de desenvolvimento estivesse em pé e funcionando. Podemos concordar que isso é bem demorado, já que para subir todo o ambiente localmente eu levaria muito menos tempo.

Com a ajuda de um amigo de trabalho começamos a procurar uma solução para esse problema, nós apanhamos bastante até descobrirmos essa página com alguns [problemas conhecidos do Docker for Mac](https://docs.docker.com/docker-for-mac/troubleshoot/#known-issues). Sendo mais preciso, o seguinte problema nos chamou a atenção:

> docker-compose 1.7.1 performs DNS unnecessary lookups for localunixsocket.local which can take 5s to timeout on some networks.

Como um último teste antes de partir para a solução sugerida desligamos todas as interfaces de rede da máquina e executamos novamente o `docker-compose start app` com as imagens cacheadas e todos os contairners subiram maravilhosamente rápido.

O problema que enfrentamos está relacionando com consultas desnecessárias ao DNS que o `docker-compose` 1.7.1 realiza e o mais interessante desse problema é exatamente a frase **em algumas redes**, pois eu uso macOS tanto no trabalho quanto em casa e na máquina do trabalho o problema era constante, já em casa o Docker funcionava perfeitamente.

Com o diagnóstico do problema partimos para a solução, nesta mesma página ele sugere que você crie uma entrada no seu `/etc/hosts` com a seguinte configuração

```
127.0.0.1 localunixsocket.local
127.0.0.1 [seu_hostname].local
```

Isso faz com que as consultas ao localunixsocket.local sejam redirecionadas para o `localhost`. Isso por si só não resolveu meu problema, então segui para o segundo passo, adicionar o seguinte comando ao seu arquivo de configuração do shell:

`docker run -d -v /var/run/docker.sock:/var/run/docker.sock -p 127.0.0.1:1234:1234 bobrik/socat TCP-LISTEN:1234,fork UNIX-CONNECT:/var/run/docker.sock &>/dev/null; export DOCKER_HOST=tcp://localhost:1234`

Isso vai criar um proxy TCP em um container que vai ser onde o Docker vai criar suas networks. Acredito que essa seja uma boa solução enquanto não resolvem o problema de forma definitiva, eu adaptei um pouco o código do comando para suprimir os erros ao executar o `docker run`, pois uma vez que o container `bobrik/socat` estiver rodando, ao iniciar uma nova sessão do shell uma excessão seria lançada pois a porta 1234 do `localhost` já está sendo utilizada.

Depois de aplicar essa correção o Docker voltou a ser utilizável no meu ambiente de desenvolvimento. Agora vocês sabem o porquê do **resolvendo problemas de *Docker for Mac* com o containers** 😋

