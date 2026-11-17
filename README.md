# VideosDGX2 â€” GeraÃ§Ã£o de VÃ­deo no DGX Spark

GeraÃ§Ã£o de vÃ­deo com **Wan 2.2** rodando localmente no **NVIDIA DGX Spark 2026**
(GB10 Blackwell Â· 128 GB de memÃ³ria unificada Â· ~1 PFLOP FP4), via **ComfyUI** e uma
interface web prÃ³pria com fila de jobs.

> ContinuaÃ§Ã£o reorganizada do repositÃ³rio original
> [`inematds/VideosDGX`](https://github.com/inematds/VideosDGX) (mantido como backup
> histÃ³rico). Aqui ficou sÃ³ o stack ativo: ComfyUI + Wan 2.2. Os pipelines antigos
> (LTX-2, Wan 2.1, MAGI-1, Waver) e o approach Docker foram removidos.

---

## Modelos ativos

| Modelo | Arquivo (em `ComfyUI/models/`) | VRAM | Uso |
|---|---|---|---|
| **Wan 2.2 14B MoE** | `unet/wan2.2_t2v_high_noise_14B_fp8_scaled.safetensors` + `â€¦_low_noise_â€¦` | ~90 GB | Qualidade mÃ¡xima (dual-expert), T2V |
| **Wan 2.2 5B** | `unet/wan2.2_ti2v_5B_fp16.safetensors` | ~25 GB | RÃ¡pido, T2V **+ I2V** |
| Text encoder (UMT5) | `text_encoders/umt5_xxl_fp16.safetensors` | â€” | Compartilhado |
| VAE | `vae/wan2.2_vae.safetensors` (5B) Â· `vae/Wan2.1_VAE.pth` (14B) | â€” | â€” |

> **Os pesos NÃƒO ficam no git** (sÃ£o ~50 GB, ignorados pelo `.gitignore`). Para baixÃ¡-los
> num ambiente novo, veja **[`doc/GUIA_COMPLETO_DO_ZERO.md`](doc/GUIA_COMPLETO_DO_ZERO.md)**.

---

## InÃ­cio rÃ¡pido

```bash
# Sobe ComfyUI (se nÃ£o estiver rodando) + a interface web
./iniciar_interface_web.sh
```

Acesse **http://localhost:7862** (ComfyUI fica em `:8188`).

Acesso remoto: descubra o IP do DGX com `hostname -I` e abra `http://<IP>:7862`
(libere a porta 7862 no firewall, se houver).

### Interface web (porta 7862)

- **Fila de jobs**: submeta vÃ¡rios vÃ­deos seguidos; processam em sequÃªncia.
- **Cancelamento**: cancela job na fila (imediato) ou em processamento.
- **I2V** (sÃ³ Wan 2.2 5B): faÃ§a upload de uma imagem inicial.
- **Presets** de resoluÃ§Ã£o/frames por modelo + parÃ¢metros avanÃ§ados (CFG, steps, split step).
- **Galeria** dos vÃ­deos gerados.

| Endpoint | MÃ©todo | DescriÃ§Ã£o |
|---|---|---|
| `/api/generate` | POST | Enfileira um job (corpo = `VideoRequest`) |
| `/api/jobs` | GET | Lista jobs + posiÃ§Ã£o na fila + progresso |
| `/api/cancel/{job_id}` | POST | Cancela um job |
| `/api/video/{filename}` | GET | Serve um vÃ­deo do output |
| `/api/all-videos` | GET | Lista a galeria |
| `/api/upload-image` | POST | Upload de imagem para I2V |

Estado da fila persistido em `/tmp/dgx_jobs_v4_2.json`.

### Linha de comando (CLI)

Cada modelo tem um gerador que submete o workflow ao ComfyUI:

```bash
# Wan 2.2 5B (T2V)
python3 gerar_video_wan22_5b.py "um gato astronauta flutuando na ISS" \
  --width 1280 --height 720 --frames 33 --fps 24 --cfg 6.0 --seed -1

# Wan 2.2 14B MoE (T2V, mÃ¡xima qualidade)
python3 gerar_video_wan22_14b.py "cidade cyberpunk Ã  noite, chuva, neon" \
  --width 1280 --height 704 --frames 81 --steps 30 --split-step 15

# Wan 2.2 5B (I2V) â€” anime a partir de uma imagem
python3 gerar_video_wan22_5b_i2v.py "a cÃ¢mera afasta lentamente" --help
```

Rode qualquer script com `--help` para todas as opÃ§Ãµes.

### Dicas de prompt

- FÃ³rmula: **`[SUJEITO] + [AÃ‡ÃƒO] + [CENÃRIO] + [ESTILO] + [QUALIDADE]`**.
- Largura/altura: mÃºltiplos de **32**.
- **Wan 2.2 14B**: mÃ¡ximo **81 frames** (acima disso aparecem artefatos) â€” detalhes em
  [`doc/WAN22_LIMITES_FRAMES.md`](doc/WAN22_LIMITES_FRAMES.md).

---

## Adicionar LTX-2.3 (opcional â€” vÃ­deo + Ã¡udio com lipsync)

O **LTX-2.3** (Lightricks, **22B**) gera **vÃ­deo e Ã¡udio sincronizados num Ãºnico modelo**,
com **lipsync nativo** e pipeline de **2 estÃ¡gios** (baixa resoluÃ§Ã£o + upscale). Ã‰ um add-on
opcional ao stack Wan 2.2 â€” **nÃ£o vem instalado** (o LTX-2 19B antigo foi removido na
reorganizaÃ§Ã£o). Hoje sÃ³ roda via **ComfyUI** (o Diffusers ainda nÃ£o suporta a 2.3).

### PrÃ©-requisitos (a mÃ¡quina jÃ¡ atende)

| Requisito | MÃ­nimo | DGX Spark |
|---|---|---|
| Python | â‰¥ 3.12 | 3.12.3 âœ… |
| CUDA | > 12.7 | toolkit 13.0 âœ… |
| PyTorch | ~2.7 | 2.10.0+cu130 âœ… |
| MemÃ³ria (22B â‰ˆ 55â€“75 GB em uso) | â€” | 128 GB unificada âœ… |
| Disco | ~80 GB (essencial) | âœ… |

### Arquivos a baixar (~80 GB essencial)

| Arquivo | Pasta (`ComfyUI/models/`) | Tamanho | Repo HF |
|---|---|---|---|
| `ltx-2.3-22b-distilled-1.1.safetensors` (recomendado, Ã¡udio melhor) | `checkpoints/` | ~46 GB | `Lightricks/LTX-2.3` |
| Gemma 3 12B QAT (text encoder, **gated**) | `text_encoders/gemma-3-12b-it-qat-q4_0-unquantized/` | ~23 GB | `google/gemma-3-12b-it-qat-q4_0-unquantized` |
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` | `loras/` | ~7.6 GB | `Lightricks/LTX-2.3` |
| `â€¦-spatial-upscaler-x2-1.1` + `â€¦-x1.5-1.0` + `â€¦-temporal-upscaler-x2-1.0` | `latent_upscale_models/` | ~3 GB | `Lightricks/LTX-2.3` |

> Para qualidade mÃ¡xima / fine-tuning, use `ltx-2.3-22b-dev.safetensors` (46 GB, bf16) no
> lugar da distilled. Kit completo (todos os checkpoints) â‰ˆ 250 GB.

### Passos

```bash
COMFYUI="$(pwd)/ComfyUI"
source comfyui-env/bin/activate

# 1. Custom node (ou via ComfyUI Manager â†’ Install/Update "LTXVideo")
git clone https://github.com/Lightricks/ComfyUI-LTXVideo "$COMFYUI/custom_nodes/ComfyUI-LTXVideo"
pip install -r "$COMFYUI/custom_nodes/ComfyUI-LTXVideo/requirements.txt"

# 2. transformers precisa estar >= 4.47.2 e < 4.52.0 (limite superior Ã© crÃ­tico)
pip show transformers

# 3. Pasta nova dos upscalers
mkdir -p "$COMFYUI/models/latent_upscale_models"

# 4. Login no HF (Gemma Ã© gated â€” aceitar a licenÃ§a no site primeiro) e baixar
#    CLI atual = `hf` (huggingface_hub recente; o antigo `huggingface-cli` foi descontinuado)
hf auth login
hf download Lightricks/LTX-2.3 ltx-2.3-22b-distilled-1.1.safetensors --local-dir "$COMFYUI/models/checkpoints/"
hf download Lightricks/LTX-2.3 ltx-2.3-22b-distilled-lora-384-1.1.safetensors --local-dir "$COMFYUI/models/loras/"
hf download Lightricks/LTX-2.3 ltx-2.3-spatial-upscaler-x2-1.1.safetensors ltx-2.3-spatial-upscaler-x1.5-1.0.safetensors ltx-2.3-temporal-upscaler-x2-1.0.safetensors --local-dir "$COMFYUI/models/latent_upscale_models/"
hf download google/gemma-3-12b-it-qat-q4_0-unquantized --local-dir "$COMFYUI/models/text_encoders/gemma-3-12b-it-qat-q4_0-unquantized/"

# 5. Reiniciar o ComfyUI e abrir um workflow de exemplo do node
```

**Notas (ARM64 + Blackwell GB10):** flash-attention e os q8 kernels compilam on-the-fly via
`ninja` (jÃ¡ funcionava no LTX-2 anterior); se a compilaÃ§Ã£o falhar, o ComfyUI cai para o
attention padrÃ£o (mais lento, mas funciona). MemÃ³ria **nÃ£o** Ã© gargalo (o 19B de 41 GB jÃ¡
rodava aqui).

---

## Estrutura do projeto

```
.
â”œâ”€â”€ web_interface.py              # Interface web (FastAPI, fila + cancel + I2V) â€” porta 7862
â”œâ”€â”€ iniciar_interface_web.sh      # Sobe ComfyUI + interface web
â”œâ”€â”€ reiniciar_comfyui.sh          # Reinicia o ComfyUI (usado entre jobs)
â”œâ”€â”€ gerar_video_wan22_14b.py      # Gerador CLI â€” Wan 2.2 14B MoE (T2V)
â”œâ”€â”€ gerar_video_wan22_5b.py       # Gerador CLI â€” Wan 2.2 5B (T2V)
â”œâ”€â”€ gerar_video_wan22_5b_i2v.py   # Gerador CLI â€” Wan 2.2 5B (I2V)
â”œâ”€â”€ workflow_wan22_14b_t2v.json   # Workflow ComfyUI â€” 14B
â”œâ”€â”€ workflow_wan22_5b_t2v.json    # Workflow ComfyUI â€” 5B
â”œâ”€â”€ requirements.txt
â”œâ”€â”€ doc/                          # DocumentaÃ§Ã£o (ver abaixo)
â”œâ”€â”€ ComfyUI/                      # App + modelos (ignorado pelo git)
â””â”€â”€ comfyui-env/                  # venv Python (ignorado pelo git)
```

## DocumentaÃ§Ã£o

| Doc | ConteÃºdo |
|---|---|
| [`doc/CONFIGURACAO_ATUAL.md`](doc/CONFIGURACAO_ATUAL.md) | Estado atual: hardware, modelos, portas, scripts |
| [`doc/GUIA_COMPLETO_DO_ZERO.md`](doc/GUIA_COMPLETO_DO_ZERO.md) | Montar tudo do zero (ComfyUI + Wan 2.2 + download dos pesos) |
| [`doc/WAN22_LIMITES_FRAMES.md`](doc/WAN22_LIMITES_FRAMES.md) | Limites de frames/duraÃ§Ã£o do Wan 2.2 |
| [`doc/research-findings.md`](doc/research-findings.md) | Pesquisa de modelos de vÃ­deo p/ DGX Spark |
| [`doc/skyreels-v3/`](doc/skyreels-v3/) | AvaliaÃ§Ã£o do candidato SkyReels V3 (+ risco) |
| [`doc/arquivo/`](doc/arquivo/) | HistÃ³rico (por que o Docker foi abandonado, plano Wan 2.2, marcos) |

---

## Hardware

- **Plataforma**: NVIDIA DGX Spark 2026
- **GPU**: Blackwell GB10 Â· **MemÃ³ria**: 128 GB unificada (CPU+GPU) Â· **~1 PFLOP FP4**
- A memÃ³ria unificada permite carregar modelos grandes sem offload para CPU; quantizaÃ§Ã£o
  FP8 Ã© usada no 14B para caber com folga.

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe