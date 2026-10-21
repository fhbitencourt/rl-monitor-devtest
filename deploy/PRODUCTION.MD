# Prontidão para produção — rl-monitor

**Última revisão:** 2026-10-20
**Responsável:** <seu nome>
**Status:** em desenvolvimento (não deployar)

Este checklist define o que precisa estar verdadeiro **antes** de o rl-monitor
tocar a rede Linux do IME. Derivado do `handoffV4.md`. Itens marcados ✅ já são
satisfeitos pelo ambiente de desenvolvimento.

Legenda:
- [ ] pendente
- [x] concluído
- [~] parcial / decisão tomada, execução pendente

---

## 1. Antes de tocar a rede do IME (Fase 0, handoff §14 e §18)

Nada aqui é opcional. Enquanto estes itens não estiverem fechados, o monitor
roda apenas no laptop, com mocks.

- [ ] Portas testadas a partir do `huginn`: `quimera:88`, `salinas:53`,
      `conti:53`, `cafezinho:25`.
- [ ] Service account LDAP `cn=rl-monitor,ou=services,dc=linux,dc=ime,dc=usp,dc=br`
      criada, read-only, bind testado em LDAPS 636 com a CA interna.
- [ ] Principal Kerberos `rl-monitor@LINUX.IME.USP.BR` criado + keytab gerado
      e testado com `kinit -kt`.
- [ ] Bot Telegram criado, chat ID obtido, mensagem de teste recebida.
- [ ] Host do stack completo decidido (`huginn` ou VM na `192.168.242.0/16`).
- [ ] Labels de sala/VLAN confirmados com a equipe.
- [ ] Datas de expiração de `kiwi.`, `wiki.`, `linux.ime.usp.br` verificadas
      e registradas (§12).
- [ ] Confirmado se `nereu` tem PBS com API ou só snapshot ZFS.
- [ ] Review do plano com a equipe de rede concluído (§19).

## 2. Secrets e credenciais (handoff §11)

- [x] `.gitignore` cobre `config.yaml`, `.env`, `*.keytab`, `*.crt`, `*.key`,
      `deploy/alertmanager/telegram_token`.
- [x] Nenhuma credencial hardcoded no código Go.
- [ ] `docker-compose.prod.yml` usa `*_FILE` / Docker secrets — sem senha em
      variável de ambiente visível em `docker inspect`.
- [ ] Fallbacks de dev removidos na config de produção (`:-admin` não vai para
      produção).
- [ ] `LDAP_BIND_PW`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `API_TOKEN`
      provisionados no host de produção com permissão `0600`, dono correto.
- [ ] Keytab com permissão `0600`, acessível apenas pelo usuário do serviço.
- [ ] Senha do break-glass do Grafana guardada no cofre da equipe (não no
      repositório, não no `.env` do host).
- [ ] Rotação de secrets documentada (quem troca, quando, como).

## 3. Exposição de rede e firewall (handoff §11)

- [ ] Prometheus e Alertmanager **não** expostos fora de `127.0.0.1` / rede
      interna do Compose.
- [ ] `node_exporter` com bind restrito (`127.0.0.1` ou interface interna).
- [ ] `blackbox_exporter` idem.
- [ ] Apenas o Grafana exposto, e apenas atrás de SSO (§11).
- [ ] Regras de firewall do host validadas com a equipe de rede.
- [ ] Portas do compose conferidas contra a política de firewall do IME.
- [ ] Nenhum container em `network_mode: host` sem justificativa registrada.

## 4. Stack de alertas (handoff §10 e §15)

- [ ] Alerta `CertificadoExpirando` (30 dias) validado **em produção**
      (não só no dev).
- [ ] Caminho Alertmanager → Telegram testado em produção com alerta real.
- [ ] `send_resolved: true` confirmado (mensagem de resolução chega).
- [ ] `group_by`, `group_wait`, `group_interval`, `repeat_interval` revisados
      com a equipe (evitar spam e evitar silêncio).
- [ ] Regras `down`/`degraded` com `down_consecutive: 3` /
      `degraded_consecutive: 1` (§15, regra 2) configuradas.
- [ ] Supressão por `root_cause` validada: dependente não alerta quando a raiz
      está down (§7 e §15, regra 5).
- [ ] Alertas de teste (silence, resolved) verificados na UI do Alertmanager.
- [ ] Runbook por alerta: o que fazer quando `CertificadoExpirando` dispara
      (§12, plano B `--webroot`).

## 5. Grafana, SSO e break-glass (handoff §11)

- [ ] Grafana atrás do Keycloak SSO.
- [ ] `disable_login_form = true` em produção.
- [ ] Admin local de emergência **existe e foi testado** com o SSO desligado
      (§15, regra 11: "o monitor tem que ver a queda que está reportando").
- [ ] Procedimento de break-glass documentado e acessível à equipe.
- [ ] Datasource Prometheus provisionado e validado.
- [ ] Dashboard provisionado a partir de `deploy/grafana/dashboards/`.
- [ ] Pasta do dashboard consistente (`rl-monitor`) — decisão de convenção.
- [ ] Backup do `grafana.db` incluído no plano de backup (§7 abaixo).

## 6. `rl-monitor` e `rl-agent` em produção (handoff §3.2, §8, §9, §15)

- [x] API não depende de LDAP/SSO para responder (§15, regra 10) — validar
      com SSO derrubado em teste.
- [ ] Decisão registrada: `rl-monitor` roda como container ou systemd no host.
- [ ] `Dockerfile.monitor` (se container) ou `rl-monitor.service` pronto e
      testado no host de produção.
- [ ] `rl-agent.service` pronto; roda como usuário não-privilegiado (§15, regra 9).
- [ ] `SupplementaryGroups=utmp` ou `who -u` — caminho escolhido e testado.
- [ ] `rl-agent` **não** expõe identidade de usuário (§8, regra imutável).
- [ ] `rl-agent` acessível apenas em localhost ou rede interna.
- [ ] Jitter ±20% ativo nos checks (§15, regra 4).
- [ ] Todo check com `context.WithTimeout` (§15, regra 3) — revisão de código.
- [ ] Prometheus scrapeia `rl-monitor` e `rl-agent` em produção (jobs
      descomentados no `prometheus.yml`).

## 7. Persistência, backup e retenção (handoff §3.4, §4)

- [ ] Retenção do Prometheus definida (30–90 dias, §1) e dimensionada em disco.
- [ ] Volume `prometheus-data` em storage com espaço suficiente e monitorado.
- [ ] Backup de `grafana.db` e `alertmanager-data` agendado.
- [ ] Backup do `config.yaml` de produção (sem secrets) versionado fora do Git.
- [ ] Restore testado pelo menos uma vez (backup que nunca foi restaurado não
      é backup).
- [ ] `config.yaml`, keytabs e certs fora do Git (§11) — confirmado no host.

## 8. Observabilidade do próprio monitor

- [ ] O monitor é monitorado: alerta se o Prometheus/Alertmanager/Grafana
      cair (dead man's switch / `watchdog` do Alertmanager).
- [ ] `node_exporter` do host do stack incluído no scrape.
- [ ] Alerta para falha de scrape (`up == 0`) do próprio stack.
- [ ] Verificação periódica de que o caminho Telegram está vivo (ex.: alerta
      sintético diário).

## 9. Privacidade por design (handoff §8 e §15)

- [ ] Nenhum campo de identidade de usuário em métrica, log ou API.
- [ ] Sessões SSH apenas como contagem (`ssh_idle_sessions`), nunca por usuário.
- [ ] Struct `AgentMetrics` sem leitura do campo `user` — revisão de código.
- [ ] Logs do `rl-agent` e `rl-monitor` sem PII (conferir amostras).
- [ ] Se Wazuh entrar no futuro, auditoria de identidade fica lá, não aqui.

## 10. Documentação e transferência (handoff §14, Fase 3)

- [ ] `README.md` com overview, arquitetura, setup, ambientes (dev vs prod).
- [ ] Seção "Black-box vs white-box vs identidade" (dúvida recorrente).
- [ ] Seção "Como monitorar certificados e o que fazer se expirar".
- [ ] Plano B de renovação via `--webroot` documentado (§12).
- [ ] Decisões de design registradas: privacy by design, correlação,
      break-glass.
- [ ] `config.example.yaml` atualizado e igual ao contrato real.
- [ ] Runbooks de alerta linkados no README.
- [ ] Alguém além de você consegue subir o stack seguindo o README
      (teste de transferência).

## 11. Deploy e rollback

- [ ] `docker-compose.prod.yml` completo e revisado.
- [ ] Deploy documentado passo a passo (comandos exatos).
- [ ] Rollback documentado (voltar versão anterior, restaurar volumes).
- [ ] Janela de manutenção combinada com a equipe (se afetar algo).
- [ ] Healthcheck dos containers configurado (`healthcheck:` no compose).
- [ ] `restart: unless-stopped` em todos os serviços de produção.
- [ ] Logs com rotação configurada (evitar encher disco).

## 12. Pós-deploy (primeiras 72h)

- [ ] Stack estável por 72h sem restart inesperado.
- [ ] Pelo menos um alerta real recebido e tratado via Telegram.
- [ ] Nenhum falso positivo recorrente.
- [ ] Uso de disco do Prometheus dentro do esperado.
- [ ] Equipe avisada de que o monitor está ativo e como silenciar alertas.
- [ ] Revisão pós-deploy agendada (o que funcionou, o que ajustar).

---

## Critério de "pronto para produção"

O rl-monitor está pronto para produção quando:

1. Todas as seções 1–3 estiverem 100% concluídas (pré-requisitos e segurança).
2. Seção 4 (alertas) concluída — alertar é o produto (§15, regra 1).
3. Seção 5 (break-glass) concluída — o monitor precisa sobreviver à queda que
   reporta.
4. Seções 7 e 11 concluídas — backup e rollback existem e foram testados.
5. Review final com a equipe de rede registrado.

Seções 8–10 e 12 podem ser concluídas nas primeiras semanas de operação, mas
não podem ficar em aberto indefinidamente.

---

## Como usar este documento

- Revise no fim de cada fase (§14 do handoff).
- Marque `[~]` quando a decisão estiver tomada mas a execução não.
- Adicione itens conforme riscos novos aparecerem; remova os que não se
  aplicarem, com justificativa no commit.
- Não deploye com itens das seções 1–3 ou 5 em aberto.
