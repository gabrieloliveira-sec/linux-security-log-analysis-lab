# Linux Security Log Analysis Lab

Laboratório de análise de logs no Ubuntu com foco em eventos de autenticação e investigação introdutória de SOC (Security Operations Center).

## Objetivo

Consultar registros do Linux, gerar uma falha de autenticação **controlada** com `sudo`, identificar a evidência nos logs e reconstruir a sequência de eventos. O exercício foi realizado em ambiente próprio, sem acesso a servidores corporativos.

## Ambiente

- **Sistema:** Ubuntu Linux em Oracle VirtualBox
- **Hostname:** `UBUNTU-LAB`
- **Conta de laboratório:** `lab`
- **Ferramentas:** `journalctl`, `grep`, `sudo`

## 1. Consultar eventos recentes

```bash
sudo journalctl -n 20 --no-pager
```

O parâmetro `-n 20` limita a saída aos 20 registros mais recentes, enquanto `--no-pager` imprime diretamente no terminal. Logs podem conter eventos de aplicativos e erros gráficos sem relação com incidentes de segurança.

## 2. Filtrar falhas de autenticação

```bash
sudo journalctl --no-pager | grep "authentication failure"
```

O operador `|` envia a saída de `journalctl` ao `grep`, que procura a expressão indicada. Na primeira consulta não houve correspondências. Isso não comprova que a máquina nunca teve incidentes; apenas indica que o texto não apareceu nos registros consultados.

## 3. Gerar uma falha controlada

```bash
sudo -k
sudo whoami
```

O comando `sudo -k` invalida a autenticação em cache. Durante a prática, uma senha incorreta foi digitada intencionalmente uma vez e, depois, a senha correta. O comando `sudo whoami` retornou `root`, indicando a identidade efetiva **daquele comando**, não uma mudança permanente de usuário.

## 4. Investigar o registro

```bash
sudo journalctl --since "10 minutes ago" --no-pager | grep -i "authentication failure"
```

O parâmetro `--since` restringe o período; `grep -i` ignora diferenças entre letras maiúsculas e minúsculas.

**Registro observado no laboratório:**

```text
Oct 19 17:13:57 UBUNTU-LAB sudo[4065]: pam_unix(sudo:auth): authentication failure; logname=lab uid=1000 euid=0 tty=/dev/pts/0 user=lab
```

| Campo | Interpretação |
| --- | --- |
| `UBUNTU-LAB` | Host que registrou o evento |
| `sudo[4065]` | Processo sudo e PID |
| `pam_unix(sudo:auth)` | Módulo PAM envolvido na autenticação |
| `authentication failure` | Falha de autenticação |
| `user=lab` | Conta associada ao evento |
| `tty=/dev/pts/0` | Terminal associado |
| `uid=1000` / `euid=0` | Identificadores registrados de usuário e usuário efetivo |

O nome de uma conta **não identifica com certeza a pessoa física** que a utilizou.

## 5. Reconstruir a linha do tempo

```bash
sudo journalctl -t sudo --since "30 minutes ago" --no-pager
```

A opção `-t sudo` seleciona mensagens identificadas com a tag `sudo`.

**Sequência observada:**

1. **17:13:57:** falha de autenticação registrada.
2. **17:14:37:** execução autorizada de `COMMAND=/usr/bin/whoami`.
3. Abertura e encerramento da sessão privilegiada do comando.

A correlação é compatível com a simulação. **Não é evidência de invasão.** Além disso, `journalctl` não registra automaticamente todos os comandos digitados no terminal.

## 6. Conclusão da investigação

**Resultado:** uma falha de autenticação local foi gerada e localizada com sucesso, seguida de uma execução autorizada de `sudo whoami`.

**Classificação:** evento esperado de laboratório, sem incidente confirmado.

Em um SOC corporativo, seria necessário correlacionar outras fontes, como histórico de acessos, origem da sessão, inventário do equipamento e registros centralizados em um SIEM. O `journalctl` executado nesta VM consulta seus próprios logs disponíveis; não acessa automaticamente os registros de outras máquinas.

## Evidências

As imagens abaixo foram registradas durante a execução do laboratório no Ubuntu e mostram as etapas realizadas. Os arquivos foram enviados à raiz do repositório com seus nomes originais.

### Evidência 1 — Simulação da falha e consulta ao journal

![Terminal Ubuntu: simulação de senha incorreta e consulta da falha de autenticação](Captura%20de%20tela%202026-10-19%20143030.png)

A captura mostra a invalidação da autenticação em cache com `sudo -k`, a tentativa de autenticação malsucedida em `sudo whoami` e a posterior consulta ao `journalctl` para localizar o evento `authentication failure`. Também aparecem tentativas anteriores de digitação de comandos corrigidas durante a prática.

### Evidência 2 — Correlação e linha do tempo do sudo

![Terminal Ubuntu: eventos sudo com falha de autenticação e execução autorizada de whoami](Captura%20de%20tela%202026-10-19%20143047.png)

A segunda captura exibe o filtro `journalctl -t sudo` e a sequência de eventos: falha de autenticação às **17:13:57**, seguida do registro de execução de `/usr/bin/whoami` às **17:14:37**, com abertura e encerramento da sessão privilegiada.

**Interpretação:** a sequência é consistente com o teste autorizado e não constitui, isoladamente, evidência de invasão.

## Próximos passos

- Anexar as capturas da falha e da linha do tempo.
- Estudar centralização e monitoramento de eventos com **Wazuh**.

---

**Projeto educacional | Linux · Log Analysis · Authentication · SOC Fundamentals**
