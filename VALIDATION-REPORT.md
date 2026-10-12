# Relatório de validação pós-auditoria

**Data:** 12 de outubro de 2026.  
**Artefato original auditado:** `ubuntu-autoinstall.zip`.  
**Ambiente local:** Debian GNU/Linux 13 (trixie), Bash, Python 3 com PyYAML 6.0.3.  
**Limite:** este host não é Ubuntu 26.04 nem o notebook de destino. Nenhum comando `--apply`, instalador, regra real de firewall/SSH, cópia para SSD real, particionamento, formatação ou criptografia foi executado.

## Testes executados após as correções

| Teste | Resultado | Evidência | Limitação | Próxima ação |
|---|---|---|---|---|
| Sintaxe Bash de todos scripts/testes | **PASSOU** | `bash tests/test-configs.sh` percorreu os `.sh` e executou `bash -n`. | Não detecta todo erro semântico nem testa todos os comandos do alvo. | Rodar ShellCheck e validar runtime no Ubuntu 26.04. |
| Sintaxe YAML e invariantes locais | **PASSOU** | `tests/validate-yaml.sh` imprimiu `PASSOU: YAML legível...`; PyYAML 6.0.3 presente. | Não substitui o JSON Schema oficial do Subiquity. | Executar `tests/validate-subiquity.sh` contra fonte oficial revisada. |
| Autoinstall sem `storage` | **PASSOU** | Teste estático não encontrou `storage:` no YAML padrão. | Não prova o comportamento na ISO Desktop específica. | Inicializar ISO 26.04 em VM UEFI e revisar a tela de armazenamento. |
| Scan heurístico de segredos | **PASSOU** | `tests/scan-secrets.sh` não encontrou marcadores conhecidos de chave privada/token nem campo sensível preenchido no YAML. | Scanner heurístico não detecta todo segredo possível; não substitui revisão manual. | Rodar antes de publicar qualquer versão e revisar mudanças de segredos manualmente. |
| Busca estática de comandos destrutivos | **PASSOU** | Padrões de `wipefs`, `mkfs`, `parted`, `sgdisk --zap`, `dd of=/dev` e `cryptsetup luksFormat` não foram encontrados no YAML/scripts. | Busca de padrões não prova ausência universal de todo código perigoso. | Revisar mudanças futuras e manter modo interativo. |
| Backup sem `rsync --delete`/remoção de origem | **PASSOU** | Teste ancorado em linhas de comando `rsync` não encontrou `--delete` nem `--remove-source-files`. | Não comprova uma cópia real. | Testar com disco descartável e verificar hashes. |
| SSH aceita CIDR local exata | **PASSOU** | Interface `ip` simulada retornou `192.168.10.20/24`; helper aceitou `192.168.10.0/24` em `--dry-run`. | Interface simulada, não a LAN do notebook. | Testar no Ubuntu 26.04 antes de aplicar. |
| SSH rejeita `0.0.0.0/0` | **PASSOU** | O helper retornou falha no cenário simulado. | Não testa `sshd` ou UFW reais. | Testar em VM isolada. |
| SSH rejeita outra sub-rede | **PASSOU** | `192.168.11.0/24` foi recusada com interface simulada `192.168.10.20/24`. | Não cobre todas as interfaces/IPv6 do hardware real. | Conferir `ip -o addr show up scope global` no alvo. |
| Backup recusa origem `/` e destino `/` | **PASSOU** | Testes executados: `/` como origem retornou `A origem / é proibida`; `/` como destino retornou `O destino / é proibido`. | Nenhum SSD real foi usado; esses testes só validam bloqueios de segurança. | Testar cópia fictícia em volume descartável. |
| `post-install.sh --dry-run` | **PASSOU** | Executado e imprimiu o plano sem alterações. | Não testa candidatos APT, Snap Store ou serviços de Ubuntu-alvo. | Simular em VM Ubuntu 26.04 com rede ativa. |
| Modos `--dry-run`/`--apply` conflitantes | **PASSOU** | `post-install.sh --dry-run --apply` e `configure-ssh-lan.sh --dry-run --apply ...` retornaram código 2; testes também verificam combinação inválida no backup. | Não testa aplicação privilegiada. | Confirmar fluxo interativo na VM. |
| Verificador reconhece VS Code Snap | **PASSOU — estático** | O teste confirma a presença do caminho `snap list code` em `verify-install.sh`. | Snap real não foi instalado neste ambiente. | Verificar `snap list code` e `code --version` em Ubuntu 26.04. |
| `storage-report.sh` somente leitura | **PASSOU** | Script executou sem alterações e listou mounts/dispositivos da VM/contêiner, incluindo `/dev/vda` e `overlay`. | Não representa o notebook; `mokutil` ausente e UEFI não detectado no ambiente. | Executar localmente e conferir modelo/capacidade/serial/UUID sem publicar dados sensíveis. |
| Validador APT de pacotes fora do alvo | **NÃO EXECUTADO — esperado (código 2)** | `tests/validate-packages.sh` reconhece Debian 13 e sai antes de consultar candidatos. | Não prova disponibilidade de todas as entradas no Ubuntu 26.04. | Executar em Ubuntu 26.04 com `main`/`universe`; habilitar `multiverse` somente se necessário para Steam. |
| ShellCheck | **NÃO EXECUTADO** | `shellcheck` não está instalado. | Não há resultado do ShellCheck. | Instalar a ferramenta numa VM de teste e rodar em `scripts/` e `tests/`. |
| Schema Subiquity oficial | **NÃO EXECUTADO** | A execução do wrapper foi interrompida pela proteção contra root (código 2); não há cópia local definida em `SUBIQUITY_REPO`. | Nenhum schema oficial foi executado neste ambiente. | Em sessão de usuário não-root, obter/revisar a cópia oficial, definir `SUBIQUITY_REPO` e executar conforme `docs/testes.md`. |
| Instalação real com ISO Desktop 26.04 em VM UEFI | **NÃO EXECUTADO** | Nenhuma VM foi inicializada/instalada nesta auditoria. | Fluxo do instalador não foi observado. | Criar VM descartável e testar conta, teclado, storage, LUKS, boot e recuperação. |
| Teclado físico ABNT2 | **DEPENDE DE HARDWARE** | YAML usa `layout: br`, `variant: abnt2`; a referência oficial define esses campos como opções XKB. | Nenhum teclado físico do usuário no ambiente de teste. | Testar `ç`, `ã`, acentos, símbolos e tela de desbloqueio. |
| UFW/AppArmor/atualizações/SSH aplicados | **NÃO EXECUTADO** | Nenhum `--apply`; no host faltam `ufw` e `sshd`. | Estado real dos serviços no Ubuntu é desconhecido. | Aplicar em VM, verificar `ufw status verbose`, `aa-status`, timers e `sshd -T`. |
| Instalação/assinatura real do Snap VS Code | **NÃO EXECUTADO** | Script migrado para método Snap oficial; o ambiente não realizou a instalação. | Rede, Store, arquitetura e publisher não foram validados por execução local. | Testar no Ubuntu 26.04; se falhar, registrar erro e manter hardening. |
| LUKS/TPM/Secure Boot/dual boot | **DEPENDE DE HARDWARE / NÃO EXECUTADO** | Nenhuma ação em disco/firmware foi feita. | Criptografia e preservação do Windows não podem ser inferidas sem VM/hardware. | Testar em VM/disco descartável, então revisar partições reais e backups. |
| Backup por UUID e recuperação | **NÃO EXECUTADO** | Só foi executado o caso proibido `/`; não houve cópia real. | Nenhum SSD externo real montado ou verificado. | Testar em volume descartável; conferir UUID, tamanho e hashes. |

## Resultado

Os testes locais estáticos e os cenários de regressão possíveis passaram. A validação APT foi corretamente marcada como não executada por estar em Debian 13. ShellCheck, schema Subiquity, instalação na ISO-alvo e testes de hardware permanecem pendentes.

**Status geral: AUDITADO E CORRIGIDO EM CÓDIGO; NÃO CERTIFICADO PARA INSTALAÇÃO DEFINITIVA.**

O Autoinstall continua sem destino de disco automático. Desconecte fisicamente o SSD externo durante a instalação. Não apague o NVMe até revisar o particionamento real, eventuais partições Windows, backup e método de criptografia.
