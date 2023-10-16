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
  > CONFIGFILE: /usr/local/samba/etc/samba/smb.conf

## Instalação do Samba

### Dependências

Para instalar as dependências em um sistema Debian-like, execute:

```
sudo apt install acl attr autoconf dnsutils bison build-essential \
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

```
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

```
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

```
samba-tool dns zonecreate 143.54.0.1 0.54.143.in-addr.arpa -U Administrator
```
O Samba deve estar iniciado para que a criação de zona seja executada.  

Crie o registro PTR de DNS (reverso) para o novo DC:

```
samba-tool dns add 143.54.0.1 0.54.143.in-addr.arpa 1 PTR dc1.samdom.sbcb.inf.ufrgs.br -U Administrator
```

## Configurando o Kerberos

Durante o provisionamento, o Samba criou um arquivo de configuração para o DC.  
Portanto, basta copiá-lo para a pasta de configuração do Kerberos no sistema.
Execute:

```
cp /usr/local/samba/private/krb5.conf /etc/krb5.conf
```

# Testando o ambiente

Para iniciar o Samba manualmente, execute:

```
samba
```

O Samba não possui scripts de System V init tais como *systemd* ou *upstart*.

## Verificando o DNS

Para verificar que o DNS do AD foi configurado corretamente, execute algumas queries DNS:

- Registro *_ldap* SRV tcp-based do domínio
  ```
  host -t SRV _ldap._tcp.samdom.sbcb.inf.ufrgs.br.
  ```
- Registro *_kerberos* SRV udp-based do domínio
  ```
  host -t SRV _kerberos._tcp.samdom.sbcb.inf.ufrgs.br.
  ```
  
# Configuração e ingresso dos clientes no domínio AD

Agora que o servidor DC AD foi configurado, precisamos configurar o Samba client nos computadores que ingressarão o domínio e efetuar o ingresso em si.
