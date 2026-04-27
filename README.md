# Investigacao-de-Incidente
## Investigação de Incidente — Initech Corp
> TryHackMe | Network Security | Perimeter Logs Challenge  
---

## Contexto

A Initech Corp, empresa de serviços financeiros, implementou recentemente um novo firewall e IDS. Analistas notaram tráfego anômalo, fui encarregado de revisar um mês de logs de perímetro e determinar se o perímetro foi violado.

**Arquivos analisados:**
- `firewall.log`
- `ids_alerts.log`
- `vpn_auth.log`
<img width="1189" height="817" alt="image" src="https://github.com/user-attachments/assets/007bfcd9-80fe-41c6-8fd6-b2b72886e787" />


**Ativos da rede:**

| IP | Host | Papel | Criticidade |
|---|---|---|---|
| 10.0.0.20 | FINANÇAS-SRV1 | Servidor de Arquivos (SMB) | Alto |
| 10.0.0.50 | VPN-GW | VPN Portal | Crítico |
| 10.0.0.51 | APP-WEB-01 | Web/Aplicativo interno | Alto |
| 10.0.0.60 | ESTAÇÃO-60 | Estação de trabalho | Médio |
| 10.8.0.23 | VPN-CLIENTE-ATTK | VPN Cliente (efêmero) | Crítico |
| 10.0.1.10 | DMZ-WEB | Servidor Web DMZ | Médio |

---

## Fase 1 — Reconhecimento (Port Scanning)

Comecei filtrando as conexões bloqueadas no firewall:

    cat firewall.log | grep "BLOCK" | head
<img width="1029" height="275" alt="image" src="https://github.com/user-attachments/assets/3763cadd-70f9-4aac-816d-65bdd4e71462" />


Resultado imediato: o IP `203.0.113.145` aparecia repetidamente tentando portas diferentes em hosts internos — 21, 22, 23, 445, 3389.

Para confirmar qual IP era responsável pela maior parte do reconhecimento:

    cat firewall.log | grep "BLOCK" | cut -d' ' -f5 | cut -d: -f1 | sort -nr | uniq -c

Resultado:
<img width="1570" height="277" alt="image" src="https://github.com/user-attachments/assets/5a76e738-4f6d-49e8-bab7-ba04ea64a8cc" />


**Conclusão:** `203.0.113.45` foi o principal IP externo realizando reconhecimento ativo contra a rede.

**Host interno mais visado:** `10.0.0.20` (FINANÇAS-SRV1)

---

## Fase 2 — Acesso inicial (Força bruta na VPN)

Com o IP suspeito identificado, fui verificar os logs de VPN:

    cat vpn_auth.log | grep FAIL | cut -d' ' -f3 | sort -nr | uniq -c
    
<img width="1305" height="166" alt="image" src="https://github.com/user-attachments/assets/84772f5c-1e07-4187-a795-bb0d87722a6f" />
Resultado: 118 tentativas de login falhas do IP suspeito em sequência, todas contra o usuário `svc_backup`.

<img width="1042" height="793" alt="image" src="https://github.com/user-attachments/assets/2e351fa4-86c0-4068-bbd6-4426f792a4ea" />

Após dezenas de falhas em intervalos de 10 segundos, o atacante obteve sucesso:

    2025-09-03 02:19:40 [REDACTED] svc_backup SUCCESS assigned_ip=10.8.0.23

**Conclusão:** Ataque de força bruta bem-sucedido contra a conta `svc_backup`. O atacante recebeu o IP interno `10.8.0.23` — agora está dentro da rede.

---

## Fase 3 — Movimento lateral

Com acesso interno via VPN, filtrei o firewall pelo IP atribuído ao atacante (`10.8.0.23`):

    cat firewall.log | grep "10.8.0.23" | grep "ALLOW" | head
<img width="1086" height="310" alt="image" src="https://github.com/user-attachments/assets/aaa1f843-7abc-4910-bdd9-4e346831d2f1" />

O host comprometido começou a sondar máquinas internas nas portas:
- **445** (SMB) → `10.0.0.20` (FINANÇAS-SRV1)
- **3389** (RDP) → múltiplos hosts
- **22** (SSH) → múltiplos hosts

Confirmado pelo IDS:

    cat ids_alerts.log | grep "10.8.0.23" | head

Alguns alertas gerados:
<img width="1800" height="598" alt="image" src="https://github.com/user-attachments/assets/0c4461a2-7575-4827-bf38-f3ddb10b1656" />

**Porta usada para movimento lateral via SMB:** `445`

**Conclusão:** O atacante se moveu lateralmente pela rede interna explorando SMB, RDP e SSH a partir do host comprometido.

---

## Fase 4 — Beacon C2 (Command & Control)

Filtrei os logs do IDS em busca de comunicação C2:

    cat ids_alerts.log | grep C2 | head

Alertas encontrados a partir de `10.0.0.60` (ESTAÇÃO DE TRABALHO-60):

 <img width="1806" height="515" alt="image" src="https://github.com/user-attachments/assets/f9388b60-cc9e-433d-bcaa-d9a5ba6696d0" />

O padrão era claro: conexões em intervalos regulares (a cada 6 horas) sempre para o mesmo IP externo e mesma porta.

**Host comprometido enviando beacon:** `10.0.0.60`  
**IP do servidor C2:** `198.51.100.77`

**Conclusão:** A estação de trabalho do setor de vendas estava com um trojan ativo, fazendo beaconing para um servidor C2 externo na porta 4444.

---

## Fase 5 — Exfiltração de dados

Com o C2 identificado, investiguei tráfego de saída do host comprometido:

<img width="1648" height="528" alt="image" src="https://github.com/user-attachments/assets/d3d5563e-29bc-4578-803e-eb1c0701288a" />
<img width="1351" height="499" alt="image" src="https://github.com/user-attachments/assets/dbcc588b-00e1-4653-ae88-c146b0f552d1" />
Os resultados mostram claramente que o host comprometido 10.0.0.51 está enviando um grande volume de tráfego para endereços IP externos.

    cat firewall.log | grep "10.0.0.51" | cut -d' ' -f5,6,7 | uniq | sort
<img width="1304" height="534" alt="image" src="https://github.com/user-attachments/assets/7c8f5019-31ca-4945-bcd4-538fe9ce1533" />

O host `10.0.0.51` (APP-WEB-01) estava enviando um volume massivo de requisições para IPs externos nas portas 80 e 8080.

Confirmado pelo IDS:

    cat ids_alerts.log | grep "10.0.0.51" | tail
<img width="1818" height="540" alt="image" src="https://github.com/user-attachments/assets/2b6a131c-648b-42a0-bd6b-a71363e79f43" />

    ET INFO Possible HTTP POST Large Upload — Classification: Potential Data Exfiltration — Priority 2

**Host com tentativas de exfiltração:** `10.0.0.51`

**Conclusão:** O servidor de aplicação interno estava exfiltrando dados via HTTP POST para servidores externos.

---
## Linha do tempo do ataque

| Data | Fase | Descrição |
|---|---|---|
| 2025-08-26 | Reconhecimento | `203.0.113.45` inicia port scanning contra hosts internos |
| 2025-09-03 | Acesso inicial | Força bruta bem-sucedida na VPN — conta `svc_backup` comprometida |
| 2025-09-05 | Movimento lateral | Host `10.8.0.23` sonda rede interna via SMB/RDP/SSH |
| 2025-09-11 | Beacon C2 | `10.0.0.60` inicia comunicação com servidor C2 em `198.51.100.77:4444` |
| 2025-09-27 | Exfiltração | `10.0.0.51` envia grandes volumes de dados para IPs externos |

---
*TryHackMe | Network Security — Perimeter Logs Challenge*
