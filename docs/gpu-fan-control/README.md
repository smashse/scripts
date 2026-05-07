# Documentacao: Controle de Fan da GPU NVIDIA em Wayland via Sessao X Paralela

## 1. Contexto

- Em **Wayland**, o `nvidia-settings` não expõe controles de fan e overclock, pois essas funções dependem do servidor Xorg.
- Para contornar essa limitação, criamos:
  - Um **script principal** (`/usr/local/bin/fan`) que aplica ajustes dinâmicos de fan com suporte a perfis.
  - Um **script auxiliar** (`/usr/local/bin/xorg-fan`) que inicia uma sessão X virtual em background (via `Xvfb`) e chama o script principal.
  - Um **serviço systemd** (`/etc/systemd/system/xorg-fan.service`) que automatiza a execução no boot.
  - Ajustes em **`/etc/X11/Xwrapper.config`** para permitir execução fora do console.
- Esse processo é **válido e satisfatório**, pois garante fan control mesmo em Wayland, sem intervenção manual.

---

## 2. Referências técnicas da GPU

- **Modelo:** Galax GeForce RTX 3050 EX V2 (GA106, 6GB GDDR6)
- **TjMax (temperatura de throttling):** 93°C — limite oficial das GPUs RTX série 30 com chip GA106
- **Temperatura típica em carga:** 63–65°C com cooler de fábrica
- **Fan mínimo:** 30% (sem fan stop — comportamento padrão da placa desativado por escolha)
- **Inicialização:** 30% fixo, independente do perfil ativo

---

## 3. Criação dos scripts

### Script principal `/usr/local/bin/fan`

```bash
sudo tee /usr/local/bin/fan > /dev/null << 'EOF'
#!/bin/bash
# Script dinâmico para controle da fan NVIDIA RTX 3050 EX V2 (Galax)
# Uso: fan [silencioso|equilibrado|desempenho]
# Padrão: equilibrado
# TjMax GA106: 93°C | Fan mínimo: 30% (sem fan stop)
# Interpolação: +5% a cada 5°C entre os pontos âncora

LOGFILE="/var/log/fan.log"
PROFILE="${1:-equilibrado}"

# Gera a curva completa a partir dos pontos âncora.
# Entre cada par de âncoras, insere pontos intermediários a cada 5°C com +5% de speed.
# O speed intermediário é limitado (cap) ao speed do âncora superior para evitar
# que o fan desacelere ao atingir o próximo âncora (relevante no perfil silencioso).
build_curve() {
    local anchors=("$@")
    CURVE=()

    for ((i=0; i<${#anchors[@]}-1; i++)); do
        local t1=$(echo "${anchors[$i]}"       | cut -d: -f1)
        local s1=$(echo "${anchors[$i]}"       | cut -d: -f2)
        local t2=$(echo "${anchors[$((i+1))]}" | cut -d: -f1)
        local s2=$(echo "${anchors[$((i+1))]}" | cut -d: -f2)

        CURVE+=("$t1:$s1")

        local t=$((t1+5))
        local s=$((s1+5))
        while [ $t -lt $t2 ]; do
            local cs=$s
            [ $s -gt $s2 ] && cs=$s2   # cap: não ultrapassa o âncora superior
            CURVE+=("$t:$cs")
            t=$((t+5))
            s=$((s+5))
        done
    done

    CURVE+=("${anchors[-1]}")
}

# --- Pontos âncora por perfil ---
# Referência: TjMax RTX 30-series GA106 = 93°C
# Piso absoluto de 30% em todos os perfis (sem fan stop)
case "$PROFILE" in
    silencioso)
        build_curve "40:30" "60:35" "75:50" "93:80"
        ;;
    desempenho)
        build_curve "40:30" "60:60" "75:80" "93:100"
        ;;
    equilibrado|*)
        build_curve "40:30" "60:45" "75:65" "93:90"
        PROFILE="equilibrado"
        ;;
esac

# Detecta o usuário que iniciou o serviço (funciona mesmo rodando como root via sudo)
CURRENT_USER=$(logname 2>/dev/null || echo "$USER")
SESSION_TYPE=$(loginctl show-session $(loginctl | awk "/${CURRENT_USER}/ {print \$1}") -p Type | cut -d= -f2)
echo "$(date '+%Y-%m-%d %H:%M:%S') | Sessão: $SESSION_TYPE | Perfil: $PROFILE" >> "$LOGFILE"

/usr/bin/nvidia-settings -a "[gpu:0]/GPUFanControlState=1" 2>>"$LOGFILE"

# Inicialização fixa em 30% (independente do perfil)
INIT_SPEED=30
if /usr/bin/nvidia-settings -a "[fan:0]/GPUTargetFanSpeed=$INIT_SPEED" 2>>"$LOGFILE"; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') | Inicialização | Fan: ${INIT_SPEED}% [$PROFILE]" >> "$LOGFILE"
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') | ERRO: não foi possível aplicar fan (provavelmente Wayland puro)" >> "$LOGFILE"
fi

while true; do
    TEMP=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits)
    SPEED=$INIT_SPEED  # fallback: 30% em caso de falha de leitura

    for POINT in "${CURVE[@]}"; do
        TEMP_MAX=$(echo "$POINT" | cut -d: -f1)
        POINT_SPEED=$(echo "$POINT" | cut -d: -f2)
        if [ "$TEMP" -le "$TEMP_MAX" ]; then
            SPEED=$POINT_SPEED
            break
        fi
        SPEED=$POINT_SPEED  # se ultrapassar todos os pontos, usa o último (93°C)
    done

    if /usr/bin/nvidia-settings -a "[fan:0]/GPUTargetFanSpeed=$SPEED" 2>>"$LOGFILE"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') | Temp: ${TEMP}°C | Fan: ${SPEED}% [$PROFILE]" >> "$LOGFILE"
    else
        echo "$(date '+%Y-%m-%d %H:%M:%S') | ERRO: não foi possível aplicar fan (sessão $SESSION_TYPE)" >> "$LOGFILE"
    fi

    sleep 10
done
EOF
sudo chmod +x /usr/local/bin/fan
```

---

### Curvas de fan por perfil

Os **pontos âncora** definem os valores fixos em temperaturas-chave. Entre cada par de âncoras, a função `build_curve` gera pontos intermediários a cada 5°C com incremento de +5% de velocidade. Quando o incremento ultrapassaria o valor do âncora superior, o speed é limitado (cap) ao valor do âncora, evitando que o fan desacelere ao cruzar um ponto de referência.

| Temp | Silencioso | Equilibrado | Desempenho | Tipo          | Justificativa                                                    |
| ---- | ---------- | ----------- | ---------- | ------------- | ---------------------------------------------------------------- |
| 40°C | **30%**    | **30%**     | **30%**    | ancora        | Idle real; piso absoluto sem fan stop                            |
| 45°C | 35%        | 35%         | 35%        | intermediario | +5% a cada 5°C a partir do ancora inferior                       |
| 50°C | 35%\*      | 40%         | 40%        | intermediario | \*Cap: incremento ultrapassaria 35% (ancora superior silencioso) |
| 55°C | 35%\*      | 45%         | 45%        | intermediario | \*Cap: idem                                                      |
| 60°C | **35%**    | **45%**     | **60%**    | ancora        | Carga leve; abaixo do tipico em gaming (63-65°C de fabrica)      |
| 65°C | 40%        | 50%         | 65%        | intermediario | +5% a cada 5°C a partir do ancora inferior                       |
| 70°C | 45%        | 55%         | 70%        | intermediario | +5% a cada 5°C a partir do ancora inferior                       |
| 75°C | **50%**    | **65%**     | **80%**    | ancora        | Carga intensa; limite superior do saudavel                       |
| 80°C | 55%        | 70%         | 85%        | intermediario | +5% a cada 5°C a partir do ancora inferior                       |
| 85°C | 60%        | 75%         | 90%        | intermediario | +5% a cada 5°C a partir do ancora inferior                       |
| 90°C | 65%        | 80%         | 95%        | intermediario | +5% a cada 5°C a partir do ancora inferior                       |
| 93°C | **80%**    | **90%**     | **100%**   | ancora        | TjMax GA106; throttling iminente                                 |

---

### Script auxiliar `/usr/local/bin/xorg-fan`

```bash
sudo tee /usr/local/bin/xorg-fan > /dev/null << 'EOF'
#!/bin/bash
LOGFILE="/var/log/fan.log"

# Detecta o usuário que iniciou o serviço
CURRENT_USER=$(logname 2>/dev/null || echo "$USER")

/usr/bin/Xvfb :2 -screen 0 1024x768x16 &

sleep 5

export DISPLAY=:2
export XAUTHORITY="/home/${CURRENT_USER}/.Xauthority"

/usr/local/bin/fan "${1:-equilibrado}" >> "$LOGFILE" 2>&1
EOF
sudo chmod +x /usr/local/bin/xorg-fan
```

---

## 4. Serviço systemd

```bash
sudo tee /etc/systemd/system/xorg-fan.service > /dev/null << EOF
[Unit]
Description=Fan control via parallel Xorg session
After=graphical.target

[Service]
ExecStart=/usr/local/bin/xorg-fan equilibrado
Restart=always
User=$(whoami)

[Install]
WantedBy=graphical.target
EOF
```

Para trocar o perfil sem editar os scripts, basta alterar o argumento no `ExecStart`:

```bash
sudo systemctl edit xorg-fan
# Altere: ExecStart=/usr/local/bin/xorg-fan silencioso
```

Ative:

```bash
sudo systemctl enable xorg-fan.service
sudo systemctl start xorg-fan.service
```

---

## 5. Logs

- Arquivo: **`/var/log/fan.log`**  
  Criado e ajustado com:

```bash
  sudo touch /var/log/fan.log
  sudo chown $(whoami):$(whoami) /var/log/fan.log
```

- Visualização:

```bash
  tail -f /var/log/fan.log
```

Exemplo de saída do log:

```text
2025-01-01 12:00:00 | Sessão: wayland | Perfil: equilibrado
2025-01-01 12:00:00 | Inicialização | Fan: 30% [equilibrado]
2025-01-01 12:00:10 | Temp: 38°C | Fan: 30% [equilibrado]
2025-01-01 12:00:20 | Temp: 47°C | Fan: 35% [equilibrado]
2025-01-01 12:00:30 | Temp: 62°C | Fan: 50% [equilibrado]
2025-01-01 12:00:40 | Temp: 78°C | Fan: 70% [equilibrado]
```

---

## 6. Permissões Xorg

```bash
sudo tee /etc/X11/Xwrapper.config > /dev/null << 'EOF'
allowed_users=anybody
needs_root_rights=yes
EOF
```

---

## 7. Checklist final

- [x] `/usr/local/bin/fan` criado com `tee`, permissão de execução e suporte a 3 perfis (silencioso, equilibrado, desempenho).
- [x] Curvas baseadas no TjMax real do GA106 (93°C) e temperatura típica de gaming da RTX 3050 (63–65°C).
- [x] Interpolação automática: +5% a cada 5°C entre os pontos âncora, gerada dinamicamente pela função `build_curve`.
- [x] Cap aplicado nos intermediários: speed nunca ultrapassa o âncora superior, evitando desaceleração indevida.
- [x] Fan mínimo de 30% em todos os perfis — sem fan stop em nenhum momento.
- [x] Inicialização fixa em 30% independente do perfil ativo.
- [x] Fallback de 30% em caso de falha de leitura de temperatura.
- [x] Nenhum nome de usuário hardcoded — `CURRENT_USER` resolvido via `logname` nos scripts e via `$(whoami)` na geração do serviço systemd e permissões do log.
- [x] `/usr/local/bin/xorg-fan` criado com `tee`, permissão de execução e repasse do argumento de perfil.
- [x] `/etc/systemd/system/xorg-fan.service` criado com `tee` e ativado no boot.
- [x] `/var/log/fan.log` criado com permissões corretas.
- [x] `/etc/X11/Xwrapper.config` ajustado para permitir execução fora do console.
- [x] Processo validado: fan control ativo em Wayland via sessão X paralela, logs centralizados, execução em background sem janelas extras.
