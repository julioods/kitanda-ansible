# kitanda-ansible

Role Ansible que corrige e padroniza o armazenamento de logs do servidor de aplicação **Kitanda**:
tira os logs de `/srv/data/log` (mesmo filesystem do PostgreSQL/Redis) e move para `/var/log/kitanda`,
com logrotate e `positions.yaml` do Promtail persistente.

## O que a role faz (tudo idempotente)

| Etapa (tag)  | Ação |
|--------------|------|
| `preflight`  | Confere se o compose existe e se há espaço livre em `/var/log` para a cópia |
| `dirs`       | Cria `/var/log/kitanda/{nginx,promtail}` |
| `migrate`    | `cp -a` dos logs antigos, uma única vez (marcador `.migrated`) |
| `compose`    | Backup `compose.yaml.bkp`, troca só o lado do host dos bind mounts, adiciona mount do Promtail, valida com `docker compose config -q` |
| `logrotate`  | `/etc/logrotate.d/kitanda` (daily, rotate 3, compress) + `nginx -s reopen` |
| `validate`   | Containers rodando, `access.log` no novo caminho, sem 429 no Promtail, uso de `/srv/data` abaixo do limite |
| `cleanup`    | **Destrutivo e desligado por padrão**: esvazia `/srv/data/log` |

Se o compose mudar, `nginx`, `promtail` e `worker` são recriados automaticamente (handler).

## Uso local

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

export KITANDA_HOST=<ip-do-servidor>

# 1) dry-run
ansible-playbook -i inventories/hml/hosts.yml playbooks/site.yml --check --diff
# 2) aplicar
ansible-playbook -i inventories/hml/hosts.yml playbooks/site.yml
# 3) só depois de validar: limpar o caminho antigo
ansible-playbook -i inventories/hml/hosts.yml playbooks/site.yml -e kitanda_logs_cleanup_old=true --tags cleanup

# rollback da configuração do compose
ansible-playbook -i inventories/hml/hosts.yml playbooks/rollback.yml
```

Se o Promtail usar um arquivo de config no host, informe em
`inventories/<env>/group_vars/kitanda_app.yml`:

```yaml
kitanda_logs_promtail_config: /opt/kitanda/promtail/config.yml
```

Isso troca `/tmp/positions.yaml` por `/tmp/promtail/positions.yaml` (evita reenvio do histórico e o 429 do Loki).

## GitHub Actions

- **CI** (`ci.yml`): `yamllint`, `ansible-lint` e `--syntax-check` em todo push/PR.
- **Deploy** (`deploy.yml`): manual (*Run workflow*), escolhe `hml`/`prod`, `check` (dry-run) ou `apply`, e opção de limpeza.

Configure em **Settings → Environments** (`hml` e `prod`) os secrets:

| Secret | Conteúdo |
|--------|----------|
| `KITANDA_HOST` | IP/host do servidor do ambiente |
| `SSH_PRIVATE_KEY` | Chave privada de um usuário de deploy |
| `SSH_KNOWN_HOSTS` | Saída de `ssh-keyscan -p 2200 <host>` (confira o fingerprint) |

No ambiente `prod`, ative **Required reviewers** para exigir aprovação antes do deploy.

## Segurança

- Nenhum IP, hostname ou chave fica no repositório; tudo vem de secrets/variáveis de ambiente.
- A limpeza se recusa a rodar sem o marcador `.migrated`.
- O repositório pode ser privado ou público, mas não commite `.vault_pass`, chaves ou inventários com IPs reais.

## Estrutura

```text
.github/workflows/   ci.yml, deploy.yml
inventories/         hml/, prod/
playbooks/           site.yml, rollback.yml
roles/kitanda_logs/  defaults, tasks, templates, handlers, meta
```
