# MultiWans - Failover e Balanceamento de Carga de Links WAN

## Descrição

O MultiWans é um script em Bash projetado para gerenciar múltiplos links de internet (WANs) em um sistema Linux. Ele implementa failover automático e redirecionamento de tráfego baseado em regras, garantindo alta disponibilidade e otimizando o uso da banda disponível. O sistema monitora a conectividade dos links, define rotas específicas para diferentes tipos de tráfego e envia alertas por e-mail em caso de falhas.

## Funcionalidades

- **Failover Automático:** Em caso de falha em um link WAN, o sistema automaticamente redireciona o tráfego para outro link disponível, garantindo a continuidade do acesso à internet.
- **Redirecionamento de Tráfego:** Permite definir regras para direcionar tipos específicos de tráfego (ex: POP3/SMTP, HTTP/HTTPS) para links WANs específicos, otimizando o uso de cada link.
- **Monitoramento de Links:** Testa a conectividade dos links WANs de forma contínua usando ping e resolução de DNS.
- **Alertas por E-mail:** Envia alertas por e-mail em caso de falhas nos links, fornecendo informações detalhadas para a equipe de TI.
- **Configuração Flexível:** Todas as configurações são armazenadas em um arquivo YAML (`config.yaml`), facilitando a personalização e a gestão do sistema.
- **Logs Detalhados:** Registra todas as ações e eventos em logs do sistema, facilitando a identificação e a solução de problemas.
- **Suporte a IPv4**: Funciona com endereçamento IPv4
- **Compatibilidade com Ipset**: Utiliza Ipset para otimizar o tratamento de grandes quantidades de redes.
- **Compatibilidade com Conntrack**: Permite gerenciar conexões rastreadas pelo sistema.
- **Integração com Scripts Auxiliares**: Possibilidade de criação de scripts auxiliares que são executados de acordo com o status (UP ou DOWN) dos links.

## Pré-requisitos

- Sistema operacional Linux (testado em Debian).
- Ferramentas:
  - `conntrack`
  - `ipset`
  - `curl`
  - `rgxg`
  - `sendemail` ou `sendEmail`
  - `yq` (versão >= 4.x)
- Acesso root.
- Múltiplas interfaces de rede conectadas a diferentes links de internet.
- Configuração prévia das interfaces de rede no SO.
- Servidor de emails configurado para envio de alertas.

## Instalação e Configuração

1. **Realizar o download do projeto para o diretório `/etc`:**

   ```console
   # wget https://github.com/cleberb/multiwans/archive/refs/heads/main.zip
   # unzip main.zip
   # mv multiwans-main /etc/multiwans
   # rm -f main.zip
   ```

2. **Adequar as permissões do diretório e arquivos:**

   ```console
   # chown -R root:root /etc/multiwans
   # chmod 750 /etc/multiwans
   # chmod 750 /etc/multiwans/scripts
   # chmod 750 /etc/multiwans/multiwans
   ```

3. **Instalar as Dependências:**
   Verificar e instalar se necessário as ferramentas: `conntrack`, `ipset`, `curl`, `rgxg`, `sendemail` ou `sendEmail`, `yq`.

   ```console
   sudo apt-get update
   sudo apt-get install conntrack ipset curl rgxg sendemail
   ```

   > **Note**
   >
   > O script instalará a versão mais recente do yq diretamente do repositório do GitHub, pois as distribuições Debian não fornecem a versão 4 nos repositórios oficiais. Isso garante que a versão correta será instalada caso o yq não esteja presente ou esteja desatualizado.

4. **Configurar o `config.yaml`:**

   - Copie o arquivo de exemplo `config.example.yaml` para `config.yaml`:

     ```console
     cp -a config.example.yaml config.yaml
     ```

   - O arquivo `config.yaml` é o coração da configuração do MultiWans. Ele permite que você defina todas as opções e regras do sistema. Edite o arquivo `config.yaml` e personalize as configurações.

   > **Note**
   >
   > O arquivo `config.example.yaml` tem a descrição, valores padrões, regras e restrições de cada parâmetro.

5. **Executar manualmente:**

   Execute o script manualmente como root (ou com `sudo`):

   ```console
   sudo /etc/multiwans/multiwans
   ```

   Para interromper pressione `CTRL+C`.

6. **Configurar script de inicialização:**

   - Copie o arquivo de serviço para o diretório do systemd:

     ```console
     sudo install -u root -g root -m 644 /etc/multiwans/multiwans.service /etc/systemd/system/
     ```

   - Edite o arquivo `multiwans.service` para garantir que todas as dependências de serviços estejam corretamente configuradas. Por exemplo, se o serviço de firewall tem um nome diferente no seu sistema ou se há outros serviços que `multiwans` depende, edite a seção `[Unit]` do arquivo `multiwans.service`:

     ```ini
     [Unit]
     Description=MultiWans Service
     After=network.target firewall.service
     Wants=firewall.service

     [Service]
     Type=simple
     ExecStart=/etc/multiwans/multiwans
     Restart=on-failure

     [Install]
     WantedBy=multi-user.target
     ```

     - Substitua `firewall.service` pelo nome correto do serviço de firewall no seu sistema, ou adicione outros serviços necessários.

   - Recarregue os arquivos de serviços do systemd:

     ```console
     sudo systemctl daemon-reload
     ```

   - Habilite o serviço para iniciar automaticamente no boot:

     ```console
     sudo systemctl enable multiwans.service
     ```

   - Inicie o serviço manualmente:

     ```console
     sudo systemctl start multiwans.service
     ```

   - Verifique o status do serviço para garantir que está funcionando corretamente:

     ```console
     sudo systemctl status multiwans.service
     ```

   > **Note**
   >
   > Certifique-se de que todas as dependências estão corretamente configuradas e que o serviço `multiwans` tem permissão de execução.

## Uso de Scripts Auxiliares

O `multiwans` permite a execução de scripts auxiliares quando os links WAN ficam online (UP) ou offline (DOWN). Esses scripts podem ser usados para realizar ações específicas, como enviar notificações especiais, ajustar configurações de rede, fazer chamadas a APIs externas, ou executar outras tarefas de manutenção.

### Estrutura dos Scripts Auxiliares

Os scripts auxiliares devem estar localizados no diretório `scripts/<iface>/<status>`, onde `<iface>` é o nome da interface de rede e `<status>` pode ser `up` ou `down`.

Exemplo de estrutura de diretórios:

```txt
multiwans/
 └── scripts/
      └── eth0/
           ├── up/
           │    └── adjust_network_up.sh
           └── down/
                └── adjust_network_down.sh
```

### Exemplo de Script Auxiliar

#### `adjust_network_up.sh`

Este script será executado quando o link WAN da interface `eth0` ficar online e ajustará as configurações de rede, como adicionar uma rota específica.

```bash
#!/bin/bash
# filepath: /etc/multiwans/scripts/eth0/up/adjust_network_up.sh

# Adicionar uma rota específica quando a interface eth0 ficar online
ip route add 192.168.1.0/24 via 10.0.0.1 dev eth0
echo "Rota para 192.168.1.0/24 adicionada via 10.0.0.1 na interface eth0"
```

#### `adjust_network_down.sh`

Este script será executado quando a interface `eth0` ficar offline e removerá a rota específica.

```bash
#!/bin/bash
# filepath: /etc/multiwans/scripts/eth0/down/adjust_network_down.sh

# Remover a rota específica quando a interface eth0 ficar offline
ip route del 192.168.1.0/24 via 10.0.0.1 dev eth0
echo "Rota para 192.168.1.0/24 removida via 10.0.0.1 na interface eth0"
```

#### Permissões

> **Warning**
>
> **IMPORTANTE:** Os scripts auxiliares devem estar na pasta `scripts/<iface>/<status>` e devem ser executáveis.

Certifique-se de que os scripts auxiliares têm permissão de execução:

```console
sudo chmod +x /etc/multiwans/scripts/eth0/up/adjust_network_up.sh
sudo chmod +x /etc/multiwans/scripts/eth0/down/adjust_network_down.sh
```

### Considerações

Os scripts auxiliares podem ter qualquer nome e são executados automaticamente pelo multiwans com base no status dos links WANS. Não é necessário configurar nada adicional no multiwans para que esses scripts sejam executados.

Para organizar a ordem de execução dos scripts, você pode prefixar os nomes dos arquivos com números. Por exemplo:

```console
00_script1.sh
01_script2.sh
99_script99.sh
```

Com esses scripts auxiliares, você pode personalizar o comportamento do `multiwans` para atender às necessidades específicas do seu ambiente.

## Arquivo de Status do Daemon

O `multiwans` utiliza um arquivo de status para registrar o estado atual dos links WAN. Este arquivo é configurado pela opção `daemon.statusFile` no arquivo de configuração `config.yaml`. O valor padrão é `/tmp/.multiwans.status`.

### Configuração

No arquivo `config.yaml`, a configuração do arquivo de status é definida da seguinte forma:

```yaml
daemon:
  statusFile: /tmp/.multiwans.status
```

### Formato do Arquivo

O formato do arquivo é simples, com cada linha representando uma interface de rede e seu status atual.

```yaml
<interface>:<status>
```

Onde `<interface>` é o nome da interface de rede e `<status>` pode ser `ON`, `OFF`, ou `UNKNOW`.

Exemplo de Arquivo de Status:

```yaml
eth0:ON
eth1:OFF
eth2:UNKNOW
```

### Utilização por Ferramentas Externas

Ferramentas de monitoramento, como o Zabbix, podem utilizar este arquivo para monitorar o status dos links WANs, garantindo maior nível de observabilidade e previsibilidade no ambiente.

## Solução de Problemas

- **Verificar os logs do sistema:** Utilize o comando `journalctl -u multiwans` para ver os logs gerados pelo sistema, se utilizado `journald` ou o comando `grep multiwans /var/log/syslog`, se utilizado `syslog`.
- **Verificar erros no arquivo `config.yaml`:** Certifique-se de que não há erros de formatação ou valores inválidos no arquivo de configuração.
- **Permissões:** Certifique-se de que o script multiwans tem permissão de execução (`chmod +x multiwans`).
- **Dependências:** Verifique se todas as dependências estão instaladas e nas versões corretas.
- **Interface de Rede:** Garanta que as interfaces de rede estão corretamente configuradas no sistema operacional e ativas.
- **Servidor de Email:** Teste a configuração do servidor de email para garantir que os alertas possam ser enviados.

## Referências técnicas do projeto

- [Links Load Balancing](https://home.regit.org/netfilter-en/links-load-balancing/)
- [Netfilter Connmark](https://home.regit.org/netfilter-en/netfilter-connmark/)
- [LARTC Multiple Links Howto](http://www.lartc.org/howto/lartc.rpdb.multiple-links.html)
- [Multi-Gateway Routing with iptables and iproute2](http://blog.khax.net/2009/11/28/multi-gateway-routing-with-iptables-and-iproute2/)
- [Multi-home Routing](https://epxx.co/artigos/multihome.html)

## Contribuindo

Contribuições são sempre bem-vindas! Sinta-se à vontade para abrir issues ou enviar pull requests.
