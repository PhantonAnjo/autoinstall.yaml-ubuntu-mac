# Auditoria independente do projeto

**Data:** 12 de outubro de 2026  
**Artefato auditado:** ZIP original `ubuntu-autoinstall.zip` (37 entradas incluindo diretórios).  
**Versão corrigida:** arquivos atualmente neste projeto; consulte `VALIDATION-REPORT.md`.  
**Escopo:** revisão dos YAML, scripts Bash, listas de pacotes, testes e documentação; testes locais sem aplicar alterações ao sistema.

## Resumo executivo

**Não foi identificado achado de gravidade CRÍTICO.** Foram encontrados achados ALTO, MÉDIO e BAIXO; os ALTO listados abaixo devem ser considerados bloqueadores para a execução da versão original.

Não foi identificado erro sintático no YAML-base nem opção de Autoinstall comprovadamente inválida pela documentação oficial consultada. Isso **não** equivale à validação pelo schema Subiquity da ISO exata: essa etapa segue pendente. Também não foi provado que qualquer pacote listado seja inexistente no Ubuntu 26.04. A validação completa dos candidatos APT precisa ser feita em Ubuntu 26.04 com os componentes de repositório realmente habilitados.

Foram encontrados problemas de segurança relevantes na implementação original, sobretudo no helper SSH, na rotina de backup e na configuração do VS Code. Eles foram corrigidos nesta versão do projeto. Nenhum comando de particionamento, formatação, criptografia ou instalação real foi executado durante esta auditoria.

## Achados concretos e correções

| ID | Gravidade | Arquivo original | Problema concreto | Correção nesta versão |
|---|---|---|---|---|
| A-01 | **ALTO** | `scripts/configure-ssh-lan.sh` | A validação só exigia que a CIDR informada contivesse algum IP local. `0.0.0.0/0` contém qualquer IPv4 local, então podia liberar TCP/22 para qualquer origem. | A CIDR precisa ser privada e coincidir exatamente com uma sub-rede numa interface ativa. Testes com `0.0.0.0/0` e sub-rede alheia são regressões obrigatórias. |
| A-02 | **ALTO** | `scripts/configure-ssh-lan.sh`, `configs/ssh/60-anjo-hardening.conf.example` | O drop-in `60-anjo-hardening.conf` podia ser lido depois de uma configuração anterior. No OpenSSH, em geral o primeiro valor obtido prevalece; escrever `PasswordAuthentication no` não prova que esse será o valor efetivo. | O drop-in passa a `00-anjo-hardening.conf`; o script valida `sshd -t` e exige por `sshd -T` os valores efetivos `PermitRootLogin no`, `AuthenticationMethods publickey`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no` e `AllowUsers anjo`. |
| A-03 | **ALTO** | `scripts/configure-ssh-lan.sh` | A versão original não comprovava que a CIDR era privada e correspondia à rede local exata, não validava a chave com `ssh-keygen` e não verificava permissões completas de `.ssh`/`authorized_keys`. Também não recusava regras SSH preexistentes e potencialmente mais amplas. | Agora valida dependências, CIDR/rede local, chave pública por `ssh-keygen -lf`, dono e modos `700`/`600`, além de recusar regras OpenSSH/22 preexistentes que não correspondam somente à CIDR confirmada. |
| A-04 | **ALTO** | `scripts/configure-ssh-lan.sh` | O rollback original removia o arquivo apenas em algumas falhas iniciais. Falhas posteriores podiam deixar regra/configuração/estado do serviço diferente do estado original. | A versão atual registra se `ssh.service` estava ativo e habilitado, tenta restaurar ambos os estados, remove o arquivo temporário e a regra criada pelo script em caso de falha. Avisos de recuperação são explícitos. Runtime de rollback ainda precisa de teste no Ubuntu-alvo. |
| A-05 | **ALTO** | `scripts/backup-to-external.sh` | A origem `/` não era recusada. A checagem de sobreposição de caminhos não bloqueia a raiz, e a versão original usava `rsync` sem `-x`; podia tentar atravessar outros pontos de montagem, inclusive o próprio SSD, e copiar muito mais dados do que o pretendido. | A origem e o destino `/` são proibidos, `rsync -x` não atravessa outros sistemas de arquivos e origem/destino são comparados por dispositivo (`MAJ:MIN`). Existe teste de regressão executado para a recusa de `/`. |
| A-06 | **ALTO** | `scripts/backup-to-external.sh` | A versão original não exigia UUID confirmado do destino nem provava que origem e destino estavam em dispositivos diferentes. Um ponto de montagem errado podia ser tratado como SSD externo. | `--apply` exige `--expected-uuid UUID`, compara UUID/`MAJ:MIN`, revalida ponto de montagem e dispositivos após a confirmação e recusa destino que deixou de corresponder. Nenhuma cópia real foi feita nesta auditoria. |
| A-07 | **ALTO** | `scripts/post-install.sh` | A versão original aplicava `ufw default deny incoming` sem revisar regras `allow` existentes. A política padrão não apaga regras explícitas; uma porta anteriormente aberta podia continuar permitida. | Antes de instalar pacotes ou mudar políticas, o script verifica `ufw show added` e aborta se encontra regras personalizadas, ou se não consegue auditá-las. Nunca apaga regras automaticamente. O firewall não foi aplicado neste ambiente. |
| A-08 | **ALTO** | `scripts/post-install.sh` | A versão original instalava chave/fonte APT para VS Code sem validar o fingerprint da chave, escrevia num keyring global, definia `Pin-Priority: 9999` e não removia repositório/pin se APT ou instalação falhassem. Por `set -e`, falha no VS Code encerrava o script antes do hardening posterior. | Foi adotado o Snap oficial documentado pelo VS Code: sem chave/fonte APT adicional. Se o Snap falhar, a falha é registrada e o hardening restante continua. O confinamento `classic` é documentado como acesso amplo, não como sandbox estrita. |
| A-09 | **MÉDIO** | `scripts/backup-to-external.sh` | `mkdir -p` reutilizava destino se nome e timestamp coincidissem; o diretório `Backups-Anjo` podia ser symlink para outro caminho; `/` não era rejeitado explicitamente como destino. | O script recusa `Backups-Anjo` como symlink/tipo inesperado, bloqueia `/` como origem e destino, não reutiliza um destino já existente e cria a pasta com `mkdir` sem `-p`. Revalida UUID/dispositivo depois da confirmação. |
| A-10 | **MÉDIO** | `scripts/post-install.sh` | `read_packages` era chamado em process substitution; falha de leitura nessa substituição podia não ser refletida pelo status de `mapfile`, deixando lista vazia. | As três listas de pacotes são verificadas como arquivos legíveis antes de `mapfile`. |
| A-11 | **MÉDIO** | Scripts de aplicação | Opções de modo conflitantes podiam ser passadas juntas; a última ocorrência prevalecia, por exemplo `--dry-run --apply`. | Os scripts pós-instalação, SSH e backup recusam combinar `--dry-run` e `--apply` na mesma chamada. |
| A-12 | **MÉDIO** | `tests/validate-packages.sh` | O teste original consultava só `packages/base.txt` e permitia consultas APT em qualquer Debian-compatível; resultado no Debian 13 não prova disponibilidade no Ubuntu 26.04. | Agora só consulta APT em Ubuntu 26.04 e percorre as listas base, auditoria e jogos. Fora do alvo, retorna código 2 e indica `NÃO EXECUTADO`. |
| A-13 | **MÉDIO** | `scripts/verify-install.sh` | Após adotar VS Code Snap, a verificação apenas por `dpkg-query` não reconheceria a instalação Snap. | O verificador reconhece instalação APT ou Snap, e exige comando `code` ou `/snap/bin/code`. |
| A-14 | **MÉDIO** | `tests/test-configs.sh` | Uma busca textual ampla podia confundir uma menção de `rsync --delete` num comentário com a opção usada numa linha de comando. | A verificação é ancorada em linhas de comando `rsync`; acrescentaram-se testes dinâmicos do helper SSH e da recusa de `/` pelo backup. |
| A-15 | **MÉDIO** | `docs/seguranca.md`, `docs/acesso-remoto.md` | UFW com entrada bloqueada pode interromper Samba, servidor de impressão e descoberta mDNS; preservar Wi-Fi/rede não significa que conexões recebidas continuarão permitidas. | A documentação distingue impressão como cliente de compartilhamento/servidor e orienta exceções estreitas, restritas à LAN conhecida. Nenhuma exceção de rede é aberta automaticamente. |
| A-16 | **BAIXO** | Documentação | Algumas instruções ainda descreviam o método APT anterior do VS Code ou usavam o placeholder genérico `SUA_REDE_CIDR`. | README e documentação foram atualizados para Snap oficial, risco de `classic` e placeholder `REDE_PRIVADA_REAL`. |

## Verificação por categoria solicitada

1. **Campos do Autoinstall:** `version`, `interactive-sections`, `locale`, `keyboard`, `timezone`, `updates` e `user-data` são coerentes com a referência oficial consultada. Não há `storage`, `late-commands` ou alvo automático de disco. `anjo` e `anjo-pc` devem ser digitados nas telas: a conta/hostname não são pré-preenchidos, pois não foi criado hash de senha.
2. **Sintaxe YAML:** carregada por PyYAML; invariantes locais conferidos. Não equivale ao JSON Schema oficial.
3. **Comandos compatíveis com Ubuntu 26.04:** scripts revisados estaticamente e testados onde possível em Debian 13. Comandos que dependem de Ubuntu/systemd/UFW/Snap não foram executados no alvo.
4. **Pacotes:** nenhum nome foi comprovado como inexistente. O catálogo oficial `resolute` confirma alguns pacotes listados, inclusive `python3-pip` e `steam-installer`; a disponibilidade de todas as entradas deve ser validada por APT em Ubuntu 26.04 com `universe`/`multiverse` conforme necessário.
5. **Repositórios/assinaturas:** o método APT original do VS Code foi removido do fluxo automático. O Snap consta na documentação oficial do VS Code e recebe atualizações pelo `snapd`; instalação real não executada aqui.
6. **Idioma/ABNT2:** `layout: br` e `variant: abnt2` correspondem às opções XKB aceitas pelo Autoinstall. Teclas físicas e layout na tela de desbloqueio ainda precisam de teste no notebook; a referência alerta para o layout primário em criptografia TPM com frase-senha.
7. **Shell quoting:** caminhos de variáveis são citados nos scripts; `bash -n` foi executado. Foram acrescentadas proteções contra symlink/caminho inseguro no backup e contra combinação ambígua dos modos. ShellCheck não foi executado pois não está instalado.
8. **`late-commands`:** não há essa seção no YAML. Logo, não existem comandos `late-commands` a auditar ou executar. A ausência é intencional para evitar comandos privilegiados extras durante o instalador.
9. **Execução como root:** scripts de aplicação verificam o SO-alvo e exigem confirmação; o backup recusa execução como root; o SSH registra e tenta restaurar o estado do serviço. Aplicação privilegiada não foi executada no Ubuntu-alvo.
10. **UFW/SSH/Samba/impressão:** não há exceções automáticas. O pós-instalação aborta se encontrar regras UFW manuais; o helper SSH só admite uma rede privada local exata. Samba, CUPS e mDNS podem precisar regras estreitas manuais.
11. **Windows/dual boot:** sem seção `storage`, a configuração não descreve operações em partições. O Windows ainda poderia ser perdido por uma decisão humana errada na interface; resumo das mudanças, backup e chave BitLocker devem ser conferidos.
12. **SSD externo:** não é referenciado pelo Autoinstall. O script de backup não formata; exige UUID e separação dos dispositivos, com nova verificação após a confirmação. O ponto de destino `/` é expressamente bloqueado.
13. **LUKS:** não há frase-senha no YAML nem uso de `cryptsetup luksFormat`. A criptografia depende do fluxo da interface escolhido e deve ser testada numa VM antes do NVMe real.
14. **Secure Boot:** nenhum script desativa Secure Boot ou instala drivers proprietários indiscriminadamente. O estado não foi verificado porque `mokutil` não está disponível neste ambiente.
15. **Arquitetura:** os pacotes APT são verificados por candidato no alvo; Steam só é oferecida se houver candidato nos repositórios habilitados. Snap, GPU e drivers dependem da arquitetura/hardware.
16. **Idempotência:** políticas UFW e atualizações são configuradas para valores definidos; backups não reutilizam destinos existentes; o VS Code Snap pode ser atualizado em reaplicação. Idempotência completa da aplicação exige teste no Ubuntu 26.04.
17. **Segredos expostos:** YAML sem senha, hash ou chave LUKS. O hash gerado manualmente e chaves SSH são sensíveis e não devem ser versionados. Não foi encontrado segredo real nas verificações executadas; a busca estática não substitui revisão manual antes de publicar o repositório.
18. **Comandos destrutivos:** o modo padrão não seleciona nem modifica discos. A busca estática não encontrou os padrões destrutivos previstos e o backup não usa `rsync --delete`; uma busca textual não prova ausência universal de todo código perigoso.
19. **Tratamento de erros:** falhas nas validações essenciais, APT, UFW e AppArmor abortam ou emitem erro explícito. A falha do VS Code é isolada para não pular o hardening restante. O rollback SSH ainda requer teste de runtime no Ubuntu-alvo.
20. **Instruções reproduzíveis:** README e comandos estão documentados. A reprodução integral depende da ISO exata, repositórios, rede, hardware, partições e VM. O `rsync` foi incluído na lista base para que o helper de backup possa ser usado após a instalação padrão. Etapas não executadas permanecem explicitamente pendentes.

## Decisões sobre armazenamento

- O Autoinstall padrão não define `storage`, destino de disco nem layout destrutivo.
- O SSD externo de arquivos importantes deve ser desconectado fisicamente durante a instalação.
- Não há modo automático para apagar o NVMe neste projeto.
- Se Windows estiver no NVMe, não escolha “usar disco inteiro”; cancele se o resumo indicar remoção das partições Windows.
- O backup para o SSD externo não substitui uma cópia independente adicional nem confirma integridade até os dados serem verificados.

## Fontes técnicas principais

- Canonical, [Autoinstall reference](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html).
- Canonical, [Autoinstall validation](https://canonical-subiquity.readthedocs-hosted.com/en/latest/howto/autoinstall-validation.html).
- Canonical, [Firewall UFW](https://documentation.ubuntu.com/server/how-to/security/firewalls/).
- OpenSSH, [sshd_config(5)](https://man.openbsd.org/sshd_config): valores lidos primeiro e ordem de includes/drop-ins.
- Microsoft, [Install Visual Studio Code on Linux](https://code.visualstudio.com/docs/setup/linux): pacote Snap oficial e atualizações em segundo plano.
- Microsoft, [Linux repository signing keys](https://learn.microsoft.com/en-us/linux/packages): famílias de chaves GPG por período de criação de repositórios.
- Ubuntu Packages: [pipx/resolute](https://packages.ubuntu.com/resolute/pipx), [lynis/resolute](https://packages.ubuntu.com/resolute/lynis), [clamav-freshclam/resolute](https://packages.ubuntu.com/resolute/clamav-freshclam), [libvulkan1/resolute](https://packages.ubuntu.com/resolute/libvulkan1) e [mesa-vulkan-drivers/resolute](https://packages.ubuntu.com/resolute/mesa-vulkan-drivers).

## Conclusão

As correções reduzem os riscos concretos encontrados no ZIP original, mas não certificam o sistema final. A etapa decisiva é executar o validador Subiquity contra a configuração e depois instalar a ISO Desktop exata em VM UEFI descartável, testando conta, teclado, LUKS, reinicialização, Secure Boot e recuperação antes de tocar no NVMe real.
