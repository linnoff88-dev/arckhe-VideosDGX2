ï»¿# VideosDGX2 ê¢â¬â GeraêÂ§êÂ£o de VêÂ­deo no DGX Spark

GeraêÂ§êÂ£o de vêÂ­deo com **Wan 2.2** rodando localmente no **NVIDIA DGX Spark 2026**
(GB10 Blackwell êÂ· 128 GB de memêÂ³ria unificada êÂ· ~1 PFLOP FP4), via **ComfyUI** e uma
interface web prêÂ³pria com fila de jobs.

> ContinuaêÂ§êÂ£o reorganizada do repositêÂ³rio original
> [`linnoff88-dev/VideosDGX`](https://github.com/linnoff88-dev/VideosDGX) (mantido como backup
> histêÂ³rico). Aqui ficou sêÂ³ o stack ativo: ComfyUI + Wan 2.2. Os pipelines antigos
> (LTX-2, Wan 2.1, MAGI-1, Waver) e o approach Docker foram removidos.

---

## Modelos ativos

| Modelo | Arquivo (em `ComfyUI/models/`) | VRAM | Uso |
|---|---|---|---|
| **Wan 2.2 14B MoE** | `unet/wan2.2_t2v_high_noise_14B_fp8_scaled.safetensors` + `ê¢â¬Â¦_low_noise_ê¢â¬Â¦` | ~90 GB | Qualidade mêÂ¡xima (dual-expert), T2V |
| **Wan 2.2 5B** | `unet/wan2.2_ti2v_5B_fp16.safetensors` | ~25 GB | RêÂ¡pido, T2V **+ I2V** |
| Text encoder (UMT5) | `text_encoders/umt5_xxl_fp16.safetensors` | ê¢â¬â | Compartilhado |
| VAE | `vae/wan2.2_vae.safetensors` (5B) êÂ· `vae/Wan2.1_VAE.pth` (14B) | ê¢â¬â | ê¢â¬â |

> **Os pesos NêÆO ficam no git** (sêÂ£o ~50 GB, ignorados pelo `.gitignore`). Para baixêÂ¡-los
> num ambiente novo, veja **[`doc/GUIA_COMPLETO_DO_ZERO.md`](doc/GUIA_COMPLETO_DO_ZERO.md)**.

---

## InêÂ­cio rêÂ¡pido

```bash
# Sobe ComfyUI (se nêÂ£o estiver rodando) + a interface web
./iniciar_interface_web.sh
```

Acesse **http://localhost:7862** (ComfyUI fica em `:8188`).

Acesso remoto: descubra o IP do DGX com `hostname -I` e abra `http://<IP>:7862`
(libere a porta 7862 no firewall, se houver).

### Interface web (porta 7862)

- **Fila de jobs**: submeta vêÂ¡rios vêÂ­deos seguidos; processam em sequêÂªncia.
- **Cancelamento**: cancela job na fila (imediato) ou em processamento.
- **I2V** (sêÂ³ Wan 2.2 5B): faêÂ§a upload de uma imagem inicial.
- **Presets** de resoluêÂ§êÂ£o/frames por modelo + parêÂ¢metros avanêÂ§ados (CFG, steps, split step).
- **Galeria** dos vêÂ­deos gerados.

| Endpoint | MêÂ©todo | DescriêÂ§êÂ£o |
|---|---|---|
| `/api/generate` | POST | Enfileira um job (corpo = `VideoRequest`) |
| `/api/jobs` | GET | Lista jobs + posiêÂ§êÂ£o na fila + progresso |
| `/api/cancel/{job_id}` | POST | Cancela um job |
| `/api/video/{filename}` | GET | Serve um vêÂ­deo do output |
| `/api/all-videos` | GET | Lista a galeria |
| `/api/upload-image` | POST | Upload de imagem para I2V |

Estado da fila persistido em `/tmp/dgx_jobs_v4_2.json`.

### Linha de comando (CLI)

Cada modelo tem um gerador que submete o workflow ao ComfyUI:

```bash
# Wan 2.2 5B (T2V)
python3 gerar_video_wan22_5b.py "um gato astronauta flutuando na ISS" \
  --width 1280 --height 720 --frames 33 --fps 24 --cfg 6.0 --seed -1

# Wan 2.2 14B MoE (T2V, mêÂ¡xima qualidade)
python3 gerar_video_wan22_14b.py "cidade cyberpunk êÂ  noite, chuva, neon" \
  --width 1280 --height 704 --frames 81 --steps 30 --split-step 15

# Wan 2.2 5B (I2V) ê¢â¬â anime a partir de uma imagem
python3 gerar_video_wan22_5b_i2v.py "a cêÂ¢mera afasta lentamente" --help
```

Rode qualquer script com `--help` para todas as opêÂ§êÂµes.

### Dicas de prompt

- FêÂ³rmula: **`[SUJEITO] + [Aêâ¡êÆO] + [CENêÂRIO] + [ESTILO] + [QUALIDADE]`**.
- Largura/altura: mêÂºltiplos de **32**.
- **Wan 2.2 14B**: mêÂ¡ximo **81 frames** (acima disso aparecem artefatos) ê¢â¬â detalhes em
  [`doc/WAN22_LIMITES_FRAMES.md`](doc/WAN22_LIMITES_FRAMES.md).

---

## Adicionar LTX-2.3 (opcional ê¢â¬â vêÂ­deo + êÂ¡udio com lipsync)

O **LTX-2.3** (Lightricks, **22B**) gera **vêÂ­deo e êÂ¡udio sincronizados num êÂºnico modelo**,
com **lipsync nativo** e pipeline de **2 estêÂ¡gios** (baixa resoluêÂ§êÂ£o + upscale). êâ° um add-on
opcional ao stack Wan 2.2 ê¢â¬â **nêÂ£o vem instalado** (o LTX-2 19B antigo foi removido na
reorganizaêÂ§êÂ£o). Hoje sêÂ³ roda via **ComfyUI** (o Diffusers ainda nêÂ£o suporta a 2.3).

### PrêÂ©-requisitos (a mêÂ¡quina jêÂ¡ atende)

| Requisito | MêÂ­nimo | DGX Spark |
|---|---|---|
| Python | ê¢â°Â¥ 3.12 | 3.12.3 ê¢Åâ¦ |
| CUDA | > 12.7 | toolkit 13.0 ê¢Åâ¦ |
| PyTorch | ~2.7 | 2.10.0+cu130 ê¢Åâ¦ |
| MemêÂ³ria (22B ê¢â°Ë 55ê¢â¬â75 GB em uso) | ê¢â¬â | 128 GB unificada ê¢Åâ¦ |
| Disco | ~80 GB (essencial) | ê¢Åâ¦ |

### Arquivos a baixar (~80 GB essencial)

| Arquivo | Pasta (`ComfyUI/models/`) | Tamanho | Repo HF |
|---|---|---|---|
| `ltx-2.3-22b-distilled-1.1.safetensors` (recomendado, êÂ¡udio melhor) | `checkpoints/` | ~46 GB | `Lightricks/LTX-2.3` |
| Gemma 3 12B QAT (text encoder, **gated**) | `text_encoders/gemma-3-12b-it-qat-q4_0-unquantized/` | ~23 GB | `google/gemma-3-12b-it-qat-q4_0-unquantized` |
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` | `loras/` | ~7.6 GB | `Lightricks/LTX-2.3` |
| `ê¢â¬Â¦-spatial-upscaler-x2-1.1` + `ê¢â¬Â¦-x1.5-1.0` + `ê¢â¬Â¦-temporal-upscaler-x2-1.0` | `latent_upscale_models/` | ~3 GB | `Lightricks/LTX-2.3` |

> Para qualidade mêÂ¡xima / fine-tuning, use `ltx-2.3-22b-dev.safetensors` (46 GB, bf16) no
> lugar da distilled. Kit completo (todos os checkpoints) ê¢â°Ë 250 GB.

### Passos

```bash
COMFYUI="$(pwd)/ComfyUI"
source comfyui-env/bin/activate

# 1. Custom node (ou via ComfyUI Manager ê¢â â Install/Update "LTXVideo")
git clone https://github.com/Lightricks/ComfyUI-LTXVideo "$COMFYUI/custom_nodes/ComfyUI-LTXVideo"
pip install -r "$COMFYUI/custom_nodes/ComfyUI-LTXVideo/requirements.txt"

# 2. transformers precisa estar >= 4.47.2 e < 4.52.0 (limite superior êÂ© crêÂ­tico)
pip show transformers

# 3. Pasta nova dos upscalers
mkdir -p "$COMFYUI/models/latent_upscale_models"

# 4. Login no HF (Gemma êÂ© gated ê¢â¬â aceitar a licenêÂ§a no site primeiro) e baixar
#    CLI atual = `hf` (huggingface_hub recente; o antigo `huggingface-cli` foi descontinuado)
hf auth login
hf download Lightricks/LTX-2.3 ltx-2.3-22b-distilled-1.1.safetensors --local-dir "$COMFYUI/models/checkpoints/"
hf download Lightricks/LTX-2.3 ltx-2.3-22b-distilled-lora-384-1.1.safetensors --local-dir "$COMFYUI/models/loras/"
hf download Lightricks/LTX-2.3 ltx-2.3-spatial-upscaler-x2-1.1.safetensors ltx-2.3-spatial-upscaler-x1.5-1.0.safetensors ltx-2.3-temporal-upscaler-x2-1.0.safetensors --local-dir "$COMFYUI/models/latent_upscale_models/"
hf download google/gemma-3-12b-it-qat-q4_0-unquantized --local-dir "$COMFYUI/models/text_encoders/gemma-3-12b-it-qat-q4_0-unquantized/"

# 5. Reiniciar o ComfyUI e abrir um workflow de exemplo do node
```

**Notas (ARM64 + Blackwell GB10):** flash-attention e os q8 kernels compilam on-the-fly via
`ninja` (jêÂ¡ funcionava no LTX-2 anterior); se a compilaêÂ§êÂ£o falhar, o ComfyUI cai para o
attention padrêÂ£o (mais lento, mas funciona). MemêÂ³ria **nêÂ£o** êÂ© gargalo (o 19B de 41 GB jêÂ¡
rodava aqui).

---

## Estrutura do projeto

```
.
ê¢âÅê¢ââ¬ê¢ââ¬ web_interface.py              # Interface web (FastAPI, fila + cancel + I2V) ê¢â¬â porta 7862
ê¢âÅê¢ââ¬ê¢ââ¬ iniciar_interface_web.sh      # Sobe ComfyUI + interface web
ê¢âÅê¢ââ¬ê¢ââ¬ reiniciar_comfyui.sh          # Reinicia o ComfyUI (usado entre jobs)
ê¢âÅê¢ââ¬ê¢ââ¬ gerar_video_wan22_14b.py      # Gerador CLI ê¢â¬â Wan 2.2 14B MoE (T2V)
ê¢âÅê¢ââ¬ê¢ââ¬ gerar_video_wan22_5b.py       # Gerador CLI ê¢â¬â Wan 2.2 5B (T2V)
ê¢âÅê¢ââ¬ê¢ââ¬ gerar_video_wan22_5b_i2v.py   # Gerador CLI ê¢â¬â Wan 2.2 5B (I2V)
ê¢âÅê¢ââ¬ê¢ââ¬ workflow_wan22_14b_t2v.json   # Workflow ComfyUI ê¢â¬â 14B
ê¢âÅê¢ââ¬ê¢ââ¬ workflow_wan22_5b_t2v.json    # Workflow ComfyUI ê¢â¬â 5B
ê¢âÅê¢ââ¬ê¢ââ¬ requirements.txt
ê¢âÅê¢ââ¬ê¢ââ¬ doc/                          # DocumentaêÂ§êÂ£o (ver abaixo)
ê¢âÅê¢ââ¬ê¢ââ¬ ComfyUI/                      # App + modelos (ignorado pelo git)
ê¢ââê¢ââ¬ê¢ââ¬ comfyui-env/                  # venv Python (ignorado pelo git)
```

## DocumentaêÂ§êÂ£o

| Doc | ConteêÂºdo |
|---|---|
| [`doc/CONFIGURACAO_ATUAL.md`](doc/CONFIGURACAO_ATUAL.md) | Estado atual: hardware, modelos, portas, scripts |
| [`doc/GUIA_COMPLETO_DO_ZERO.md`](doc/GUIA_COMPLETO_DO_ZERO.md) | Montar tudo do zero (ComfyUI + Wan 2.2 + download dos pesos) |
| [`doc/WAN22_LIMITES_FRAMES.md`](doc/WAN22_LIMITES_FRAMES.md) | Limites de frames/duraêÂ§êÂ£o do Wan 2.2 |
| [`doc/research-findings.md`](doc/research-findings.md) | Pesquisa de modelos de vêÂ­deo p/ DGX Spark |
| [`doc/skyreels-v3/`](doc/skyreels-v3/) | AvaliaêÂ§êÂ£o do candidato SkyReels V3 (+ risco) |
| [`doc/arquivo/`](doc/arquivo/) | HistêÂ³rico (por que o Docker foi abandonado, plano Wan 2.2, marcos) |

---

## Hardware

- **Plataforma**: NVIDIA DGX Spark 2026
- **GPU**: Blackwell GB10 êÂ· **MemêÂ³ria**: 128 GB unificada (CPU+GPU) êÂ· **~1 PFLOP FP4**
- A memêÂ³ria unificada permite carregar modelos grandes sem offload para CPU; quantizaêÂ§êÂ£o
  FP8 êÂ© usada no 14B para caber com folga.

---

Projeto  - ARCKHÉe