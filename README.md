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
