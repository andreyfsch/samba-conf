# Configurando o Samba 4 como Active Directory Domain Controller

Para configurar um servidor de usuários, devemos atribuir a um servidor o papel de **Active Directory** (AD) **Domain Controller** (DC).
A seguir, serão fornecidas as instruções para levantar um Samba como DC e construir uma nova floresta AD.

## Preparando o ambiente

- Definia um domínio DNS para a floresta AD:
  - A utilização do domínio top-level da organização não é recomendável, sendo prudente a seleção de um subdomínio. Por exemplo: considerando o domínio top-level SBCB.INF.UFRGS.BR, um domínio possível para a floresta AD seria SMB.SBCB.INF.UFRGS.BR;
- Defina de um hostname para o AD DC que deve ter menos de 15 caracteres. Por exemplo: DC1;
- Configure um IP estático para o DC e reserve o endereço associado no roteador;
  - **Importante:** o DC se tornará o DNS resolver para todos computadores que ingressarem no domínio. Como resultado, pode ser exigido que o endereço IP esteja fora do pool DHCP do roteador.
- Desative ferramentas como **resolvconf** que automaticamente editam o arquivo de configuração de DNS resolver **/etc/resolv.conf**;
- Certifique-se de que o arquivo /etc/hosts contém o fully-qualified domain name (FQDN) e o hostname curto do endereço IP da LAN do DC. Por exemplo:
  ```
  127.0.0.1     localhost  
  143.54.48.16     DC1.smb.sbcb.inf.ufrgs.br     DC1
  ```
  O hostname e o FQDN não devem apontar para **127.0.0.1** ou para nenhum outro endereço além daquele usado na interface LAN do DC;
- Remova qualquer instância prévia do arquivo **smb.conf**. Para listar o caminho do arquivo:
  ```
  smbd -b | grep "CONFIGFILE"
  ```
  Caso um arquivo exista, um exemplo de output seria:
  ```console
  CONFIGFILE: /usr/local/samba/etc/samba/smb.conf
  ```

## Instalação do Samba

### Dependências

Para instalar as dependências em um sistema Debian-like, execute:

```console
sudo apt install acl attr autoconf dnsutils bison build-essential asciidoctor \
  debhelper dnsutils docbook-xml docbook-xsl flex gdb libjansson-dev krb5-user \
  libacl1-dev libaio-dev libarchive-dev libattr1-dev libblkid-dev libbsd-dev \
  libcap-dev libcups2-dev libgnutls28-dev libgpgme-dev libjson-perl \
  libldap2-dev libncurses5-dev libpam0g-dev libparse-yapp-perl \
  libpopt-dev libreadline-dev nettle-dev perl perl-modules-5.34 pkg-config \
  python3-all-dev python3-cryptography python3-dbg python3-dev \
  python3-dnspython python3-gpg python3-markdown \
  python3-dev xsltproc zlib1g-dev liblmdb-dev lmdb-utils libsystemd-dev
```

### Pacotes core

Para instalar o Samba, execute:

```console
sudo apt install acl attr samba winbind libpam-winbind libnss-winbind \
 krb5-config krb5-user dnsutils python3-setproctitle ntp
```

Sendo que o Kerberos realm no nosso exemplo seria:

```
SMB.SBCB.INF.UFRGS.BR
```

E o hostname solicitado para o servidor Kerberos e para o servidor administrativo no nosso exemplo seria:

```
DC1
```

## Provisionamento

Para provisionar o AD, utilizaremos o utilitário **samba-tool**.  
Substitua *PASSWORD* pela senha do administrador:

```console
samba-tool domain provision --server-role=dc --use-rfc2307 --dns-backend=SAMBA_INTERNAL --realm=SMB.SBCB.INF.UFRGS.BR --domain=SMB --adminpass=PASSWORD
```

## Configurando o DNS resolver

Edite o arquivo **/etc/resolv.conf** e insira as linhas abaixo.  
Substitua *143.54.48.16* pelo endereço IP do DC:

```
search smb.sbcb.inf.ufrgs.br
nameserver 143.54.48.16
```

## Criando uma zona reversa

Para criar uma zona de pesquia reversa, execute:

```console
samba-tool dns zonecreate 143.54.48.16 48.54.143.in-addr.arpa -U Administrator
```
O Samba deve estar iniciado para que a criação de zona seja executada.  

Crie o registro PTR de DNS (reverso) para o novo DC:

```console
samba-tool dns add 143.54.48.16 48.54.143.in-addr.arpa 1 PTR dc1.smb.sbcb.inf.ufrgs.br -U Administrator
```

## Configurando o Kerberos

Durante o provisionamento, o Samba criou um arquivo de configuração para o DC.  
Portanto, basta copiá-lo para a pasta de configuração do Kerberos no sistema.
Execute:

```console
cp /usr/local/samba/private/krb5.conf /etc/krb5.conf
```

## Configurando o serviço de sincronização horária

A recomendação oficial do Samba é a utilização do chrony, portanto seguiremos com a instalação e configuração do mesmo.  
Clone o repositório do chrony em qualquer localização do servidor executando:
```console
git clone https://gitlab.com/chrony/chrony.git
```

Entre na pasta executando:
```console
cd chrony
```

A compilação do chrony exige um compilador C, sendo a opção default o gcc.  
A configuração do utilitário *configure* instalará por default o chrony no caminho **/usr/local**. Caso se deseje alterar este caminho, deve-se passar como parâmetro para o utilitário *--prefix=/path/to/local*.  
Prosseguiremos com a configuração do chrony utilizando somente o parâmetro *--enable-ntp-signd*, obrigatório para uso no Samba, executando:
```console
./configure --enable-ntp-signd
```
A seguir, execute:
```console
make
```
Em seguida, execute:
```console
make install
```  
Agora, realizaremos a configuração do chrony, iniciando pelas permissões de socket do DC. Execute:
```
chown root:_chrony /usr/local/samba/var/lib/ntp_signd/
chmod 750 /usr/local/samba/var/lib/ntp_signd/
```
Finalmente, devemos substituir o arquivo **/etc/chrony.conf**. Utilize [o arquivo chrony.conf no  repo](chrony-server.conf) e , se necessário, edite os IPs.


# Testando o ambiente

Para iniciar o Samba manualmente, execute:

```console
samba
```

O Samba não possui scripts de System V init tais como *systemd* ou *upstart*.

## Verificando o DNS

Para verificar que o DNS do AD foi configurado corretamente, execute algumas queries DNS:

- Registro SRV *_ldap* tcp-based do domínio:
  ```console
  host -t SRV _ldap._tcp.smb.sbcb.inf.ufrgs.br.
  ```
  Exemplo de output esperado:  
  ```console
  _ldap._tcp.smb.sbcb.inf.ufrgs.br has SRV record 0 100 389 dc1.smb.sbcb.inf.ufrgs.br.
  ```
- Registro SRV *_kerberos* udp-based do domínio:
  ```console
  host -t SRV _kerberos._tcp.smb.sbcb.inf.ufrgs.br.
  ```
  Exemplo de output esperado:  
  ```console
  _kerberos._udp.smb.sbcb.inf.ufrgs.br has SRV record 0 100 88 dc1.smb.sbcb.inf.ufrgs.br.
  ```
- Registro A do DC:
  ```console
  host -t A dc1.smb.sbcb.inf.ufrgs.br.
  ```
  Exemplo de output esperado:  
  ```console
  dc1.smb.sbcb.inf.ufrgs.br has address 143.54.48.16
  ```
- Registro PTR do DC:
  ```console
  host -t PTR 143.54.48.16.
  ```
  Exemplo de output esperado:  
  ```console
  96.50.54.143.in-addr.arpa domain name pointer dc1.smb.sbcb.inf.ufrgs.br
  ```

## Verificando o Kerberos

Para este teste, solicite um tíquete Kerberos para o administrador de domínio.  
Execute:

```console
kinit administrator
```
Será solicitada a senha do administrador no formato:  
```Password for administrator@SMB.SBCB.INF.UFRGS.BR:```

Após, liste os tíquetes Kerberos em cache executando:

```console
klist
```
Exemplo de output esperado:  
```console
Ticket cache: FILE:/tmp/krb5cc_0  
Default principal: administrator@SMB.SBCB.INF.UFRGS.BR  
  
  Valid starting       Expires              Service principal  
  01.11.2016 08:45:00  12.11.2016 18:45:00  krbtgt/SMB.SBCB.INF.UFRGS.BR@SMB.SBCB.INF.UFRGS.BR  
  renew until 02.11.2016 08:44:59
```

## Autenticação de usuários de domínio utilizando PAM
Para permitir que usuários de domínio se loguem localmente ou se autentiquem em serviços instalados no membro do domínio, como SSH, o PAM deve ser capaz de utilizar o módulo *pam_winbind*.

### Configurando os parâmetros do Windbindd no smb.conf

Adicione as seguintes linhas na sessão **[global]** do arquivo *smb.conf*:
```console
template shell = /bin/bash
template homedir = /home/%U
```

### Localizando e configurando a lib libnss_winbind.so.2
Para localizar a lib que será configurada, execute:
```console
smbd -b | grep LIBDIR
```
Que resultará em um output semelhante a:
```console
LIBDIR: /usr/local/samba/lib/
```
A seguir, faça os links simbólicos que seguem e execute o comando **ldconfig**. Execute (utilizando o path obtido anteriormente):
```console
ln -s /usr/local/samba/lib/libnss_winbind.so.2 /lib/x86_64-linux-gnu/
ln -s /lib/x86_64-linux-gnu/libnss_winbind.so.2 /lib/x86_64-linux-gnu/libnss_winbind.so
ldconfig
```

### Configurando o Name Service Switch (NSS)

É necessário configurar o NSS para que usuários e grupos do domínio estejam disponíveis localmente.  
Edite o arquivo **/etc/nsswitch.conf**, adicionando *windbind* nos databases **passwd** e **group**, além de alterar a ordem da diretiva *dns* na database **hosts** da seguinte maneira:
```console
passwd: files winbind
group:  files winbind
hosts:  files dns mdns4_minimal [NOTFOUND=return] myhostname
```

Não utilize nomes de usuários iguais no domínio e no arquivo **/etc/passwd** local.

### Serviço Winbindd

Não inicialize manualmente o serviço **winbindd** no DC AD


  
# Configuração e ingresso dos clientes no domínio AD

Agora que o servidor DC AD foi configurado, precisamos configurar o Samba client nos computadores que ingressarão o domínio e efetuar o ingresso em si.

## Preparo geral

- Verifique se algum processo do Samba está rodando com a execução do comando:
  ```console
  ps ax | egrep "samba|smbd|nmbd|winbindd"
  ```

  No caso do output do comando listar algum dos processos **samba**, **smdb**, **nmdb** ou **windbindd**, finalize-os.
- Se o cliente já possuía o Samba instalado antes desse prodedimento:
  - Faça backup do arquivo de configuração **smb.conf**. Para encontrar a localização do arquivo, execute:
    ```console
    smbd -b | grep "CONFIGFILE"
    ```
    O output deverá ser semelhante a:
    ```console
    CONFIGFILE: /usr/local/samba/etc/samba/smb.conf
    ```
  - Remova todos databases do Samba, como arquivos *.tdb e *.ldb. Para listar as pastas contendo os arquivos, execute:
    ```console
    smbd -b | egrep "LOCKDIR|STATEDIR|CACHEDIR|PRIVATE_DIR"
    ```
    Exemplo de output esperado:
    ```console
    LOCKDIR: /usr/local/samba/var/lock/
    STATEDIR: /usr/local/samba/var/locks/
    CACHEDIR: /usr/local/samba/var/cache/
    PRIVATE_DIR: /usr/local/samba/private/
    ```
## Preparo do membro de domínio para ingresso no domínio AD

### Configurando o DNS

O Samba utiliza o DNS no background para localizar DCs e serviços como o Kerberos. Portanto, membros e servidores do domínio AD devem ser capazes de resolver zonas do DNS do AD.
Se o ambiente possuir um servidor DHCP provendo configurações DNS para os computadores clientes, o servidor DHCP deve ser configurado para enviar os endereços IP do servidor DNS.  
  
Configurando manualmente o cliente para utilizar o servidor DNS:
- Edite o arquivo **/etc/resolv.conf** com o endereço IP do servidor DNS e com o nome do domínio:
  ```
  nameserver 143.54.48.16
  nameserver 8.8.8.8
  search smb.sbcb.inf.ufrgs.br
  ```
- Alguns utilitários, como NetworkManager podem sobrescrever mudanças manuais no arquivo **etc/resolv.conf**.  
  No caso do NetworkManager, altere na sessão *[main]* do arquivo **/etc/NetworkManager/NetworkManager.conf** a diretiva *dns*, setando o valor para *none*.
  Um exemplo de sessão *[main]* para o arquivo seria:
  ```
  [main]
  plugins=ifupdown,keyfile
  dns=none
  ```
- Além disso, devemos desabilitar o serviço **systemd-resolved**, executando os seguintes comandos:
  ```console
  sudo systemctl stop systemd-resolvd
  sudo systemctl disable systemd-resolvd
  ```

### Testando a resolução DNS
Execute:
- Forward lookup:
  ```console
  nslookup DC1.smb.sbcb.inf.ufrgs.br
  ```
  Deve ter um output semelhante a:
  ```console
  Server:         143.54.48.16
  Address:        143.54.50.100#53
  
  Name:   DC1.smb.sbcb.inf.ufrgs.br
  Address: 143.54.48.16
  ```
- Reverse lookup:
  ```console
  nslookup 143.54.48.16
  ```
  Deve ter um output semelhante a:
  ```console
  Server:        143.54.48.16
  Address:	10.99.0.1#53

  16.48.54.143.in-addr.arpa	name = DC1.smb.sbcb.inf.ufrgs.br.
  ```
- Registros SRV:
  ```console
  host -t SRV _ldap._tcp.smb.sbcb.inf.ufrgs.br
  ```
  Deve ter um output semelhante a:
  ```console
  _ldap._tcp.smb.sbcb.inf.ufrgs.br has SRV record 0 100 389 dc1.smb.sbcb.inf.ufrgs.br.
  ```
### Configuração do Kerberos

Edite o arquivo **/etc/krb5.conf** com as seguintes diretivas:

```
[libdefaults]
  default_realm = SMB.SBCB.INF.UFRGS.BR
  dns_lookup_realm = false
  dns_lookup_kdc = true
[plugins]
  localauth = {
    module = winbind:/usr/lib64/samba/krb5/winbind_krb5_localauth.so
    enable_only = winbind
  }
```
Verifique se o arquivo possui a diretiva *include* e, caso positivo, você **deve** removê-la.

Edite também o arquivo **/etc/security/pam_winbind.conf** com a seguinte diretiva:
```
[global]
  krb5_auth = yes
  krb5_ccache_type = FILE
```

## Configurando a sincronização horária
Instale o chrony seguindo as mesmas etapas [descritas anteriormente](configurando-o-serviço-de-sincronização-horária), porém não utilize o parâmetro *--enable-ntp-signd* na configuração.  
Após, substitua o arquivo **/etc/chrony/chrony.conf** pelo [arquivo indicado no repo](chrony-client.conf), substituindo os endereços IP se necessário.

## Resolução de hostname local
Quando um host ingressa o domínio, o Samba tenta registrar o hostname na zona AD DNS. Para isso, o utilitário *net* precisa conseguir resolver o hostname utilizando DNS ou um registro correto no arquivo **/etc/hosts**.
Para verificar que a resolução do hostname está correta, execute:
```console
getent hosts DM1
```
O output deve ser semelhante a:
```console
143.54.50.100      DM1.smb.sbcb.inf.ufrgs.br    DM1
```
Certifique-se de que o arquivo **/etc/hosts** contém apenas a linha:
```console
127.0.0.1      localhost
```
O FQDN não deve aparecer neste arquivo, pois estamos utilizando DHCP para obter nosso IP.

## Instalação do Samba

Proceda para a instalação do samba e suas dependências [realizando o procedimento descrito anteriormente](#instalação-do-samba).  
A única diferença é que ao invés de **DC1** como hostname utilizado no kerberos, utilizaremos **DM1**, referente a *Domain Member*.

## Preparando o smb.conf

Para localizar o arquivo smb.conf após a instalação, execute:
```console
smbd  -b | grep CONFIGFILE
```
Sendo o output semelhante a:
```console
CONFIGFILE: /etc/samba/smb.conf
```
Agora, renomeie o arquivo original e faça download do [o arquivo smb.conf repo](smb.conf).
Execute:
```console
mv /etc/samba/smb.conf /etc/samba/smb.conf.old
wget -O /etc/samba/smb.conf https://raw.githubusercontent.com/andreyfsch/samba-conf/main/smb.conf
```
A seguir, recarregue o samba, executando:
```console
smbcontrol all reload-config
```

## Mapeando o administrador de domínio ao usuário root local

Para executar operações de arquivo no sistema de arquivos local como root, o Samba permite mapear usuários do domínio para usuários locais.  
Já que a sessão *[global]* do arquivo **smb.conf** do nosso repo já possui a diretiva ```username map = /usr/local/samba/etc/user.map```, podemos seguir para o próximo passo.  
Vamos mapear o usuário de domínio *Administartor* para o usuário local *root*, executando:

```console
echo "!root = SMB\Administrator" >> /usr/local/samba/etc/user.map
```

## Ingressando o host no domínio

Para ingressar o host no AD, execute:

```console
samba-tool domain join smb.sbcb.inf.ufrgs.br MEMBER -U administrator
```

Entre com a senha do usuário *Administrator* a seguir.  

### Configurando o Name Service Switch (NSS)

É necessário configurar o NSS para que usuários e grupos do domínio estejam disponíveis localmente.  
Edite o arquivo **/etc/nsswitch.conf**, adicionando *windbind* nos databases **passwd** e **group**, além de alterar a ordem da diretiva *dns* na database **hosts** da seguinte maneira:
```console
passwd: files winbind
group:  files winbind
hosts:  files dns mdns4_minimal [NOTFOUND=return] myhostname
```

Se houver uma linha contendo a diretiva **initgroups**, adicione ```[success=continue] winbind``` ao final, mas não crie uma linha com a diretiva se ela não existir.

Não utilize nomes de usuários iguais no domínio e no arquivo **/etc/passwd** local.

### Modidique o Kerberos start ID no PAM
```console
sudo sed -i 's/minimum_uid=1000/minimum_uid=10000/' /etc/pam.d/common-account
sudo sed -i 's/minimum_uid=1000/minimum_uid=10000/' /etc/pam.d/common-auth
sudo sed -i 's/minimum_uid=1000/minimum_uid=10000/' /etc/pam.d/common-password
sudo sed -i 's/minimum_uid=1000/minimum_uid=10000/' /etc/pam.d/common-session
sudo sed -i 's/minimum_uid=1000/minimum_uid=10000/' /etc/pam.d/common-session-noninteractive
```
Além disso, autorize a criação de diretórios home, executando o comando:
```console
sudo pam-auth-update --enable mkhomedir
```

## Inicializando serviços do Samba

Execute:
```console
sudo smbd
sudo nmbd
sudo winbindd
```
Não inicialize o serviço **samba** no membro do domínio, este serviço deve ser inicializado somente no AD DC.

## Testando a conectividade do Winbindd

### Enviando um Ping Winbindd

Para verificar que um serviço windbindd pode se conectar aos AD DC, execute:
```console
wbinfo --ping-dc
```
Que deve ter um output semelhante a:
```console
checking the NETLOGON for domain[SAMDOM] dc connection to "DC.SMB.SBCB.INF.UFRGS.BR" succeeded
```
Se o comando falhar, verifique se o serviço do **winbindd** está rodando e que as configurações no arquivo **smb.conf** estão corretas.

## Modificando o greeter do sistema operacional para login no startup
```console
sudo apt install lightdm-gtk-greeter
sudo apt purge slick-greeter
```
Reinicie o sistema após esta operação e faça login com um usuário *SMB\user*.

## Utilizando usuários e grupos do domínio em comandos do sistema operacional

### Buscando por usuários e grupos do domínio

A lib *libnss_winbind* permite a busca de usuários e grupos do domínio, por exemplo:  
- Para buscar o usuário de domínio *SMB\demo01*, execute:
  ```console
  getent passwd SMB\\demo01
  ```
  Que resultará em um output semelhante a:
  ```console
  SMB\demo01:*:10000:10000:demo01:/home/demo01:/bin/bash
  ```
- Para buscar o grupo do domínio *Domain Users*, execute:
  ```console
  getent group "SMB\\Domain Users"
  ```
  Que resultará em um output semelhante a:
  ```console
  SMB\domain users:x:10000:
  ```
### Designando permissões de arquivo para usuários e grupos do domínio

A lib **NSS** permite que se utilize contas e grupos do domínio em comandos. Por exemplo, para determinar a propriedade de um arquivo para o usuário de domínio *demo01*, e o grupo para o grupo de domínio *Domain Users*, execute:
```console
chown "SMB\\demo01:SAMDOM\\domain users" file.txt
```
