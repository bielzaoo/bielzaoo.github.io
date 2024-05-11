+++
title = '[HTB] Return'
date = 2024-05-11T01:32:41-03:00
draft = true
+++

# [HTB] Return

![htb-icon](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/htb.png?raw=true)

Opa! Eai, tudo certo?

Essa máquina é bem bacana e tranquila para quem tem dificuldades como exploração de ambientes Windows (Como eu, hehe), sabemos que encarar o Windão nos CTF da vida não é tão fácil assim. Mas enfim, chega de enrolação e vamos pro hacking!

## Conhecendo o alvo
Não podemos sair atitando sem mirar e conhecer o território inimigo. Então vamos começar conhecendo quais portas estão abertas no nosso alvo:

![Nmap1](https://raw.githubusercontent.com/bielzaoo/bielzaoo.github.io/main/images/return-htb/nmap1.png)

Muitas portas! Maaas, vamos focar nas que mais importam para nós (Vai por mim, as outras não vão te levar a lugar nenhum, bem, não neste cenário hehe), e lá vamos nós, mais um scan:

![nmap2](https://raw.githubusercontent.com/bielzaoo/bielzaoo.github.io/main/images/return-htb/nmap2.png)

Ok, agora temos uma visão mais apurada do que está rodando no máquina!

Eu gosto de fragmentar o scan desta forma, isso me possibilita ter uma visão geral dos serviços.

## Printer Admin Panel
Dando uma olhada no que está rodando na porta 80, vemos que se trata de um "Painel Administrativo de Impressora", interessante!

![site](https://raw.githubusercontent.com/bielzaoo/bielzaoo.github.io/main/images/return-htb/site.png)

Dando uma clicada na aba *settings* para ver no que dá, vemos que parace que podemos alterar as configurações do serviço, interessante!

![settings](https://raw.githubusercontent.com/bielzaoo/bielzaoo.github.io/main/images/return-htb/settings-site.png)

## Ganhando acesso ao alvo
Bom, um usuário nós já temos: *svc-printer*.  Anota ai!

Agora precisamos da senha. Você pode pensar: ah! A senha está ali, basta ir no código fonte e alterar o atributo `type` da tag `input` de *password* para *text* e a senha será mostrada! Sinto em lhe dizer, mas se você pensou assim, conseguiram te enganar, acontece hehe!

Veja o source da página:

![source](https://raw.githubusercontent.com/bielzaoo/bielzaoo.github.io/main/images/return-htb/settings-site.png)

Pois é, os asteriscos foram colocados ali hardcoded.

Maaas, não desanimemos! Veja, o serviço está usando o LDAP e o scan anterior confirma que a porta do LDAP está aberta. E, dependendo da forma como o LDAP estiver configurado, ele expõe as credenciais ao se autenticar no servidor.

Ok! Com isso em mente, vamos fazer o seguinte: iniciaremos o netcat ouvindo na mesma porta usada pelo LDAP (porta 389) na nossa máquina local, e lá no site, na opção *server address* colocaremos o IP a nossa máquina atacante, de forma que quando clicarmos em *update* no site o serviço que está sendo executado na máquina irá tentar se autenticar no nosso servidor LDAP falso, expondo assim as credenciais.

Vamos que vamos!

Já subi o netcat, e fiz a alteração lá no site (lembre-se, altere *server address* para o ip da sua máquina), após isso, cliquei em *update*, e voltando lá no netcat, veja que deu bom:

![ldap credentials](https://raw.githubusercontent.com/bielzaoo/bielzaoo.github.io/main/images/return-htb/ldap-creds.png)

Beleza, mas e agora? Onde usaremos essas credênciais?

Simples, vimos também no scan que o WinRM está rodando na máquina! Pois então vamos tentar usar essas credenciais!

Usando o `Evil-WinRM`, conseguimos  acessar a máquina:

![winrm](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/winrm-login.png?raw=true)

Beleza! Agora precisamos upar nossos privilégios. Vamos dar uma olhada em quais grupos nosso usuário pertence:

![whoami-groups](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/whoami-groups.png?raw=true)

Ah! Vemos que fazemos parte do grupo *Server Operators*, tá, mas eai?

Este grupo, basicamente, nos da permissão configurar serviços no servidor! Tá, mas eai? (De novo hehe)

Isso quer dizer que podemos escolher um serviço que seja executado como System no servidor e fazermos com que este serviço nos ajude a elevar nossos privilégios. Simplemente iremos alterar o atributo *binPath* do serviço VSS (Este é um servio padrão do sistema que roda como System, justamente o que precisamos) fazendo-o apontar para o netcat que faremos upload para o servidor e então nos dê uma reverse shell como System no servidor.

Caso ainda não tenha entendido, não se preocupe, veremos agora como funciona!

> Estou usando a **Pwnbox** disponibilizada pelo HTB, como você já deve ter notado pelo meu prompt. Na pwnbox existe um diretório que contém o netcat para Windows. O Kali Linux também tem, se eu não me engano. Caso você esteja usnado uma outra distro que não seja estas, uma rápida pesquisada "netcat windows" ou "nc64.exe" no Google, vocẽ já consegue achar.

Vamos fazer upload do netcat:

![netcat-upload](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/upload-ncat.png?raw=true)

Beleza, agora vamos alterar o binPath do serviço VSS.

Mas antes, podemos ver se realemente esse serviço roda como System, veja:

![sc-vss](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/sc-vss.png?raw=true)

Em `SERVICE_START_NAME` vemos lá *LocalSystem*.

Bom, agora podemos alterar o binPath deste serviço:

![binpath-nc](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/binpath_alterado.png?raw=true)

Agora, basta iniciar o netcat na nossa máquina local  ouvindo na mesma porta usada no comando acima (7777, no caso), após iniciado o netcat, vamos reiniciar o serviço para que o comando seja executado:

![sc-stop-start](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/sc-stop-start.png?raw=true)

Ao voltarmos la no netcat, vemos que recebemos a conexão:

![net-connection](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/ncat-connection.png?raw=true)

Porém, como visto na imagem, a shell recebida não é nada estável. rapidamente somos desconectados. Isso se deve ao fato de que os executáveis usados em serviços devem ser especialmente preparados para isso, portanto, executáveis "normais" sofrem problemas neste caso.

Há duas soluções para uma shell mais estável, vamos lá!

## Sem o Metasploit
Podemos conseguir uma shell mais estável sem o metasploit, alterando o binPath da seguinte forma:

![binPath2](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/binpath-updated2.png?raw=true)

Desta forma, estamos executando o netcat usando o cmd.exe, de forma que quando o processo do cmd.exe morrer (Lembre-se que serviços não conseguem exeeutar muito bem executáveis que não foram especialmente criados para serem serviços) o netcat permanecerá executando.

Veja que não cai mais o conexão:

![stable](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/shell-system-stable.png?raw=true).

## Com o Metasploit
Podemos usar o Metasploit para conseguir estabilidade e ainda conseguirmos uma shell meterpreter.

Começaremos criando um executável usando o `msfvenom`:

![gen-payload](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/gen-payload.png?raw=true)

Agora, basta fazermos upload dele para o alvo:

![upload-biel](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/upload-biel.png?raw=true)

Agora, precisamos alterar o binPath do serviço de forma que ele aponte para o nosso payload:

![binpath-biel](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/binpath-biel.png?raw=true)

Vamos configurar o Metasploit para receber a conexão e já deixá-lo esperando por conexões:

![metasploit-conf](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/metasploit-conf.png?raw=true)

Basta agora, reiniciar o serviço como feito anteriormente (`sc.exe stop vss` e `sc.exe start vss`), e recebemos então a conexão:

![session](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/sessions-meterpreter.png?raw=true)

E veja que já somos System. Porém ainda não acabou, se você demorar muito verá que a conexão cairá. Para que isso não ocorra, precisamos migrar para um processo que execute como System para manter nossos privilégios de System e seja mais estável.

Listando os processos:

![ps](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/ps.png?raw=true)

Vemos o `svchost`, executa como System e também é estável. Beleza! Anotando o seu PID, vamos migrar para ele:

![migrate](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/migrate.png?raw=true)

E veja que ainda somos System e a conexão está estável!

## Flags

E aqui estão as flags:

![flgas](https://github.com/bielzaoo/bielzaoo.github.io/blob/main/images/return-htb/user-root-flag.png?raw=true)

### É isso!
Bom, te espero no próximo writeup!

Fique com Deus!
