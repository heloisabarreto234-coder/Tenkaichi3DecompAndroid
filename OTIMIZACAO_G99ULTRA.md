# Otimização para Redmi Note 14 4G (MediaTek Helio G99 Ultra)

Build otimizado especificamente para o hardware:
- **CPU**: 2× Cortex-A76 @ 2.2GHz + 6× Cortex-A55 @ 2.0GHz (ARMv8.2-A, ambos suportam FP16 e Dot Product)
- **GPU**: Mali-G57 MC2
- **RAM**: LPDDR4X (6GB / 8GB)
- **SO**: Android 14+ / HyperOS

## Alterações aplicadas no código-fonte

### 1. `port/tools/android.py` — flags de compilação (o ganho principal)

O build original usava `-O2` genérico (target `aarch64-linux-android` sem tuning de CPU).
Agora o código é compilado com otimização direcionada ao Cortex-A76 (núcleo de performance,
onde 90% do engine roda), compatível com ambos os clusters:

| Código | Flags originais | Flags otimizadas | Por quê |
|---|---|---|---|
| Game original (SOFT / soft-float) | `-O2` | `-O3 -march=armv8.2-a -mtune=cortex-a76` | Melhor agendamento de instruções para o A76; `-mgeneral-regs-only` continua ativo (float do PS2 preservado) |
| LLC (IR → objeto) | `-O2` | `-O3` | Otimização mais agressiva no código do jogo |
| Port layer (HARD — gráficos/som/IO) | `-O2` | `-O3 -march=armv8.2-a+fp16+dotprod -mtune=cortex-a76 -ffp-contract=fast -fno-math-errno` | Usa FP16 e instruções de dot product (A76 e A55 suportam); FMA liberado no código de renderização |
| C++ do port (imgui/etc.) | `-O2` | mesmo conjunto HARD | Idem |

**Segurança**: `-ffp-contract=off` foi MANTIDO no código original do jogo (SOFT) para
preservar a matemática exata do PS2. O fast-math só foi aplicado no código do port
(renderização, som, I/O), onde FMA é seguro e mais rápido. Nenhuma extensão ARMv8.1+
(LSE etc.) foi ativada — ambos os clusters (A76 e A55) executam o binário sem instrução ilegal.

### 2. `android/app/src/main/AndroidManifest.xml`
- Adicionado `android:largeHeap="true"` (reduz pressão de GC no launcher/Compose).

### 3. `compile_flags.txt`
- Atualizado para refletir as novas flags (clangd / IDE).

## Perfil de GPU (já otimizado de fábrica)

O launcher já detecta **Mali-G57** como tier 0 (o mais leve) em `GpuProfiles.kt`:
- Resolução interna: **1× (scale4=4)** — nativo do PS2, ideal para a Mali-G57 MC2
- `frame_skip=1` (pula quadros em vez de reduzir a velocidade)
- `fx_off=48` (desliga ambos os tipos de motion blur)
- `frag_depth=0` (sem profundidade precisa — mais rápido)
- `interp=0` (sem interpolação 60fps — não sobrecarrega a GPU)
- Filtro: bilinear (0). **Recomendado**: se quiser mais nitidez, teste FSR (filtro=3)
  nas configurações do jogo — renderiza a 1× e faz upscale com custo baixo.

## Configurações recomendadas no app (Redmi Note 14 4G)

1. **Resolução interna**: 1× (automático no primeiro início)
2. **Renderer**: Automático (OpenGL ES para Mali — mais estável que Vulkan nesta GPU)
3. **Modo de apresentação**: Automático (mailbox/immediate = menor latência)
4. **Vsync**: desligado (o padrão automático já evita vsync onde possível)
5. **Som**: mantenha ligado; o Helio G99 tem hardware de áudio dedicado
6. **Durante o jogo**: remova capas grossas (o G99 Ultra faz throttling após ~15min
   em carga máxima; o resfriamento passivo do Note 14 é limitado). Jogue em superfície
   rígida, não na cama/travesseiro.
7. **Não carregue enquanto joga**: carga + jogo = calor excessivo = throttling de até 30%.

## Compilando

Siga o README original (toolchain via `port/tools/android_toolchain.sh`, depois
`port/tools/android.py`). As flags otimizadas são aplicadas automaticamente.

---
*Otimização direcionada a Helio G99 Ultra / Mali-G57 MC2. Compatível apenas com
arm64-v8a (o build já é exclusivo desta ABI).*
