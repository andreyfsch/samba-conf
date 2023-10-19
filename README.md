# Configurando o Samba 4 como Active Directory Domain Controller

Para configurar um servidor de usuários, devemos atribuir a um servidor o papel de **Active Directory** (AD) **Domain Controller** (DC).
A seguir, serão fornecidas as instruções para levantar um Samba como DC e construir uma nova floresta AD.

## Preparando o ambiente

- Definia um domínio DNS para a floresta AD:
  - A utilização do domínio top-level da organização não é recomendável, sendo prudente a seleção de um subdomínio. Por exemplo: considerando o domínio top-level SBCB.INF.UFRGS.BR, um domínio possível para a floresta AD seria SAMDOM.SBCB.INF.UFRGS.BR;
- Defina de um hostname para o AD DC que deve ter menos de 15 caracteres. Por exemplo: DC1;
- Configure um IP estático para o DC e reserve o endereço associado no roteador;
  - **Importante:** o DC se tornará o DNS resolver para todos computadores que ingressarem no domínio. Como resultado, pode ser exigido que o endereço IP esteja fora do pool DHCP do roteador.
- Desative ferramentas como **resolvconf** que automaticamente editam o arquivo de configuração de DNS resolver **/etc/resolv.conf**;
- Certifique-se de que o arquivo /etc/hosts contém o fully-qualified domain name (FQDN) e o hostname curto do endereço IP da LAN do DC. Por exemplo:
  ```
  127.0.0.1     localhost  
  143.54.0.1     DC1.samdom.sbcb.inf.ufrgs.br     DC1
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
  python-all-dev python-crypto python3-dbg python-dev python-dnspython \
  python3-dnspython python3-gpg python-markdown python3-markdown \
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
SAMDOM.SBCB.INF.UFRGS.BR
```

E o hostname solicitado para o servidor Kerberos e para o servidor administrativo no nosso exemplo seria:

```
DC1
```

## Provisionamento

Para provisionar o AD, utilizaremos o utilitário **samba-tool**.  
Substitua *PASSWORD* pela senha do administrador:

```console
samba-tool domain provision --server-role=dc --use-rfc2307 --dns-backend=SAMBA_INTERNAL --realm=SAMDOM.SBCB.INF.UFRGS.BR --domain=SAMDOM --adminpass=PASSWORD
```

## Configurando o DNS resolver

Edite o arquivo **/etc/resolv.conf** e insira as linhas abaixo.  
Substitua *143.54.0.1* pelo endereço IP do DC:

```
search samdom.sbcb.inf.ufrgs.br
nameserver 143.54.0.1
```

## Criando uma zona reversa

Para criar uma zona de pesquia reversa, execute:

```console
samba-tool dns zonecreate 143.54.0.1 0.54.143.in-addr.arpa -U Administrator
```
O Samba deve estar iniciado para que a criação de zona seja executada.  

Crie o registro PTR de DNS (reverso) para o novo DC:

```console
samba-tool dns add 143.54.0.1 0.54.143.in-addr.arpa 1 PTR dc1.samdom.sbcb.inf.ufrgs.br -U Administrator
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
Finalmente, devemos substituir o arquivo **/etc/chrony/chrony.conf**. Utilize [o arquivo chrony.conf no  repo] e , se necessário, edite os IPs.


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
  host -t SRV _ldap._tcp.samdom.sbcb.inf.ufrgs.br.
  ```
  Exemplo de output esperado:  
  ```console
  _ldap._tcp.samdom.sbcb.inf.ufrgs.br has SRV record 0 100 389 dc1.samdom.sbcb.inf.ufrgs.br.
  ```
- Registro SRV *_kerberos* udp-based do domínio:
  ```console
  host -t SRV _kerberos._tcp.samdom.sbcb.inf.ufrgs.br.
  ```
  Exemplo de output esperado:  
  ```console
  _kerberos._udp.samdom.sbcb.inf.ufrgs.br has SRV record 0 100 88 dc1.samdom.sbcb.inf.ufrgs.br.
  ```
- Registro A do DC:
  ```console
  host -t A dc1.samdom.sbcb.inf.ufrgs.br.
  ```
  Exemplo de output esperado:  
  ```console
  dc1.samdom.sbcb.inf.ufrgs.br has address 143.54.0.1
  ```
- Registro PTR do DC:
  ```console
  host -t PTR 143.54.0.1.
  ```
  Exemplo de output esperado:  
  ```console
  1.0.54.143.in-addr.arpa domain name pointer dc1.samdom.sbcb.inf.ufrgs.br
  ```

## Verificando o Kerberos

Para este teste, solicite um tíquete Kerberos para o administrador de domínio.  
Execute:

```console
kinit administrator
```
Será solicitada a senha do administrador no formato:  
```Password for administrator@SAMDOM.SBCB.INF.UFRGS.BR:```

Após, liste os tíquetes Kerberos em cache executando:

```console
klist
```
Exemplo de output esperado:  
```console
Ticket cache: FILE:/tmp/krb5cc_0  
Default principal: administrator@SAMDOM.SBCB.INF.UFRGS.BR  
  
  Valid starting       Expires              Service principal  
  01.11.2016 08:45:00  12.11.2016 18:45:00  krbtgt/SAMDOM.SBCB.INF.UFRGS.BR@SAMDOM.SBCB.INF.UFRGS.BR  
  renew until 02.11.2016 08:44:59
```
  
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
  nameserver 143.54.0.1
  search samdom.sbcb.inf.ufrgs.br
  ```
  Alguns utilitários, como NetworkManager podem sobrescrever mudanças manuais no arquivo **etc/resolv.conf**.  
  No caso do NetworkManager, configure o servidor DNS via GUI ou nmcli e reinicie o serviço do NetworkManager.
- 

