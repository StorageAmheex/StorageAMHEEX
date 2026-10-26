# AMHEEX Storage API v2 — Documentação Técnica

> Armazenamento em nuvem de alta performance, escrito em PHP e **gerido e processado inteiramente pelo AMHEEX SERVER**.

**URL base:** `https://storage.amheex.workers.dev`

---

## Sumário

1. [Introdução e conceito](#1-introdução-e-conceito)
2. [Novidades da v2](#2-novidades-da-v2)
3. [Como funciona: particionamento](#3-como-funciona-particionamento)
4. [Requisitos do sistema](#4-requisitos-do-sistema)
5. [Autenticação e cabeçalhos](#5-autenticação-e-cabeçalhos)
6. [Endpoint de upload](#6-endpoint-de-upload)
7. [Streaming e visualização (rotas `/X/`)](#7-streaming-e-visualização-rotas-x)
8. [Download automático (rotas sem `/X/`)](#8-download-automático-rotas-sem-x)
9. [Erros e códigos de status HTTP](#9-erros-e-códigos-de-status-http)
10. [Exemplos por linguagem](#10-exemplos-por-linguagem)

---

## 1. Introdução e conceito

A **AMHEEX Storage API** é um serviço de armazenamento backend em PHP, robusto e de alta performance, **gerido e processado pelo AMHEEX SERVER**. Ela suporta arquivos de até **1 TiB** usando particionamento inteligente em blocos binários.

Você envia o arquivo com um único `POST` e recebe de volta a URL pública. Essa URL funciona em players de vídeo, `<img>`, iframes e clientes HTTP, com streaming por `Range` (`206 Partial Content`). Para baixar o arquivo, basta tirar o `/X/` do caminho.

| Recurso | Valor |
|---|---|
| Tamanho máximo por arquivo | 1 TiB (1.099.511.627.776 bytes) |
| Tamanho de cada bloco | 25 MB (26.214.400 bytes) |
| Streaming | Range Requests → `206 Partial Content` |
| Resposta do upload | Exatamente uma URL em `text/plain` |
| CORS | Liberado para qualquer origem |

---

## 2. Novidades da v2

| Antes (v1) | Agora (v2) |
|---|---|
| Upload em partes manuais com `Content-Range` | **Um único POST** com o arquivo inteiro; o AMHEEX SERVER particiona sozinho |
| Retomar exigia consultar `?status=1` | **Retomada automática**: basta reenviar o mesmo POST |
| `409 Conflict` quando o arquivo novo era diferente do anterior | **Sobrescrita automática**: partes obsoletas são apagadas |
| Download forçado com `?download=1` | **Rota sem `/X/`** → download imediato |
| `?player=1`, `?embed=1`, `?check_size=1` | Rota `/X/` serve tudo; tamanho via `HEAD` (`Content-Length`) |
| Resposta de upload variável | **Exatamente uma URL pura** em `text/plain` |

---

## 3. Como funciona: particionamento

O AMHEEX SERVER divide cada arquivo em blocos binários de tamanho fixo e os remonta de forma transparente na entrega.

| Item | Valor |
|---|---|
| Tamanho de cada parte | **26.214.400 bytes** (25 MB exatos) |
| Máximo de partes por arquivo | 41.944 |
| Recebimento | Streaming de bytes via `php://input` (sem multipart) |

### Fluxo de um upload

1. **POST recebido**: o AMHEEX SERVER lê `X-File-Size` e calcula quantas partes de 26.214.400 bytes o arquivo terá.
2. **Verificação**: confere quais partes daquele caminho já existem no servidor.
3. **Gravação**: os bytes são gravados parte a parte. Se a última parte existente estiver incompleta, ela é completada antes de seguir.
4. **Limpeza**: se o arquivo novo for menor que o anterior, as partes excedentes são apagadas.
5. **Resposta**: com 100% montado, retorna `200 OK` e a URL pública em `text/plain`.

### Comportamento adaptativo

| Situação | O que o AMHEEX SERVER faz |
|---|---|
| Conexão caiu no meio do upload | No próximo POST, detecta as partes existentes, completa a última incompleta e continua das restantes. |
| Arquivo novo **menor** que o anterior | Sobrescreve e **apaga as partes obsoletas** excedentes, evitando corrupção do binário. |
| Arquivo novo **igual ou maior** | Sobrescreve de forma limpa e automática. |

---

## 4. Requisitos do sistema

Requisitos do servidor que executa a AMHEEX Storage:

- **PHP 8.1+ (64-bit)**, necessário para endereçar arquivos acima de 2 GB
- **Extensão cURL** habilitada
- **AMHEEX SERVER** configurado

Do lado do cliente, qualquer ferramenta HTTP serve: navegador, curl, Node.js, Python, PHP etc.

---

## 5. Autenticação e cabeçalhos

A autenticação com a camada de armazenamento é feita **internamente pelo AMHEEX SERVER**. O cliente não envia token: basta mandar os cabeçalhos abaixo.

| Cabeçalho | Uso | Descrição |
|---|---|---|
| `X-File-Size` | **Obrigatório** no upload | Tamanho **total** do arquivo em bytes. |
| `Content-Type` | Recomendado no upload | `application/octet-stream`: o corpo são os bytes crus. |
| `Range` | Opcional no GET | Ex.: `bytes=0-1048575` → resposta `206`. |

> ⚠️ O `X-File-Size` precisa ser **exatamente** o tamanho do arquivo. É com ele que o AMHEEX SERVER calcula as partes, detecta retomadas e apaga partes obsoletas.

### CORS

Todas as rotas respondem com `Access-Control-Allow-Origin`, e o preflight (`OPTIONS` → `204`) libera `X-File-Size`. Ficam legíveis no JavaScript: `Content-Length`, `Content-Range`, `Accept-Ranges`, `Content-Type`, `Content-Disposition`, `ETag` e `Last-Modified`.

---

## 6. Endpoint de upload

```
POST /X/{caminho/arquivo.ext}
```

| Item | Descrição |
|---|---|
| Corpo | Bytes crus do arquivo (stream). Não use `multipart/form-data`. |
| `X-File-Size` | Obrigatório: tamanho total em bytes. |
| `Content-Type` | `application/octet-stream` |
| Resposta `200` | `text/plain` com **exatamente uma URL**: o link público direto. |

**Resposta de sucesso:**

```http
HTTP/1.1 200 OK
Content-Type: text/plain

https://storage.amheex.workers.dev/X/pasta/video.mp4
```

### cURL

```bash
# Upload (streaming direto do disco — recomendado para arquivos grandes)
curl -X POST \
  -H "Content-Type: application/octet-stream" \
  -H "X-File-Size: $(stat -c%s video.mp4)" \
  -T video.mp4 \
  https://storage.amheex.workers.dev/X/pasta/video.mp4

# Resposta (text/plain) — exatamente a URL pública:
# https://storage.amheex.workers.dev/X/pasta/video.mp4
```

> 💡 Prefira `-T arquivo` a `--data-binary @arquivo`: o `-T` envia direto do disco, enquanto o `--data-binary` carrega tudo na memória antes.

### cURL com retomada automática

```bash
# Upload com retomada automática: se a conexão cair, o curl reenvia
# e o AMHEEX SERVER continua a partir das partes que já existem.
curl -X POST --retry 10 --retry-all-errors --retry-delay 3 \
  -H "Content-Type: application/octet-stream" \
  -H "X-File-Size: $(stat -c%s backup.tar.gz)" \
  -T backup.tar.gz \
  https://storage.amheex.workers.dev/X/backups/backup.tar.gz
```

### JavaScript (Fetch)

```javascript
// Navegador — upload com Fetch API
const API = "https://storage.amheex.workers.dev";

async function enviar(file, pasta = "uploads") {
  const url = `${API}/X/${pasta}/${encodeURIComponent(file.name)}`;
  const res = await fetch(url, {
    method: "POST",
    headers: {
      "Content-Type": "application/octet-stream",
      "X-File-Size": String(file.size),
    },
    body: file,
  });
  if (!res.ok) throw new Error(`Upload falhou: HTTP ${res.status}`);
  return (await res.text()).trim(); // URL pública pura
}

// Uso
document.querySelector("#arquivo").addEventListener("change", async (e) => {
  const link = await enviar(e.target.files[0]);
  console.log("Disponível em:", link);
});
```

### JavaScript (XHR com progresso e retomada)

```javascript
// Navegador — upload com XMLHttpRequest (barra de progresso + retomada)
const API = "https://storage.amheex.workers.dev";

function enviar(file, onProgress, tentativas = 5) {
  const url = `${API}/X/uploads/${encodeURIComponent(file.name)}`;
  return new Promise((resolve, reject) => {
    const tentar = (restantes) => {
      const xhr = new XMLHttpRequest();
      xhr.open("POST", url);
      xhr.setRequestHeader("Content-Type", "application/octet-stream");
      xhr.setRequestHeader("X-File-Size", String(file.size));
      xhr.upload.onprogress = (e) => e.lengthComputable && onProgress(e.loaded / e.total);
      xhr.onload = () => xhr.status === 200
        ? resolve(xhr.responseText.trim())          // URL pública pura
        : reject(new Error(`HTTP ${xhr.status}`));
      // Conexão caiu? Reenvie: o AMHEEX SERVER retoma das partes existentes.
      xhr.onerror = () => restantes > 0
        ? setTimeout(() => tentar(restantes - 1), 3000)
        : reject(new Error("Falha de rede"));
      xhr.send(file);
    };
    tentar(tentativas);
  });
}

// Uso
enviar(arquivo, (p) => (barra.value = Math.round(p * 100)))
  .then((link) => console.log("Disponível em:", link));
```

### Retomada após queda de conexão

Não precisa calcular offsets: **reenvie o mesmo POST**, com o mesmo caminho e o mesmo `X-File-Size`. O AMHEEX SERVER verifica as partes já gravadas, completa a última se estiver incompleta e continua das restantes.

---

## 7. Streaming e visualização (rotas `/X/`)

```
GET  /X/{caminho/arquivo.ext}
HEAD /X/{caminho/arquivo.ext}
```

| Requisição | Resposta |
|---|---|
| Sem `Range` | `200`: binário completo, exibido inline |
| Com `Range` | `206 Partial Content` + `Content-Range` |
| `HEAD` | Só cabeçalhos: `Content-Length`, `Content-Type`, `Accept-Ranges` |

O AMHEEX SERVER entrega o binário puro, remontado a partir das partes de 25 MB. É a rota ideal para players de vídeo e áudio, `<img>`, iframes e requisições HTTP comuns.

```html
<!-- Vídeo com streaming nativo (o player usa Range automaticamente) -->
<video controls preload="metadata" src="https://storage.amheex.workers.dev/X/pasta/video.mp4"></video>

<!-- Imagem -->
<img src="https://storage.amheex.workers.dev/X/pasta/foto.jpg" alt="Foto">

<!-- Iframe -->
<iframe src="https://storage.amheex.workers.dev/X/pasta/documento.pdf" width="100%" height="600"></iframe>
```

```bash
# Streaming: primeiros 1 MiB (resposta 206 Partial Content)
curl -H "Range: bytes=0-1048575" -D - -o trecho.bin \
  https://storage.amheex.workers.dev/X/pasta/video.mp4

# Somente metadados (tamanho, tipo, Accept-Ranges)
curl -I https://storage.amheex.workers.dev/X/pasta/video.mp4
```

---

## 8. Download automático (rotas sem `/X/`)

```
GET /{caminho/arquivo.ext}
```

Retorna o binário com `Content-Disposition: attachment`: o navegador **baixa imediatamente**.

### `/X/` ou sem `/X/`?

| | **Com `/X/`** | **Sem `/X/`** |
|---|---|---|
| Exemplo | `https://storage.amheex.workers.dev/X/pasta/video.mp4` | `https://storage.amheex.workers.dev/pasta/video.mp4` |
| Comportamento | Exibe no navegador (inline) | Força o download |
| Streaming / Range | ✅ `206` | — |
| Uso ideal | Players, `<img>`, iframes, APIs | Botões "Baixar", links de entrega |

```bash
# Download automático (Content-Disposition: attachment) — rota sem /X/
curl -L -O -J https://storage.amheex.workers.dev/pasta/video.mp4
```

```html
<!-- O navegador baixa o arquivo imediatamente -->
<a href="https://storage.amheex.workers.dev/pasta/relatorio.pdf">Baixar relatório</a>
```

---

## 9. Erros e códigos de status HTTP

| Código | Significado | Quando acontece / o que fazer |
|---|---|---|
| `200` | OK | Upload 100% montado (corpo = URL pública) ou arquivo entregue por completo. |
| `204` | No Content | Resposta ao preflight CORS (`OPTIONS`). |
| `206` | Partial Content | GET com `Range` nas rotas `/X/`. |
| `302` | Found | Arquivo não encontrado: redireciona para `/erro/404.html`. Confira o caminho. |
| `400` | Bad Request | `X-File-Size` ausente, inválido ou diferente do corpo enviado. |
| `405` | Method Not Allowed | Método não suportado na rota. |
| `413` | Payload Too Large | Arquivo excede 1 TiB. |
| `416` | Range Not Satisfiable | `Range` fora do tamanho do arquivo. |
| `5xx` | Erro no servidor | Falha temporária no AMHEEX SERVER. Reenvie o mesmo POST: o upload retoma das partes existentes. |

### Boas práticas

- Só considere o upload concluído com `200` e uma URL no corpo.
- Em falhas de rede ou `5xx`, repita o mesmo POST com espera crescente (3 s, 6 s, 12 s…).
- Para saber se um arquivo existe e qual o tamanho, faça `HEAD` na rota `/X/`.

```javascript
// Tratamento de erros recomendado (navegador ou Node.js)
async function baixarInfo(url) {
  const res = await fetch(url, { method: "HEAD" });

  // Arquivo inexistente: o AMHEEX SERVER redireciona para /erro/404.html
  if (res.redirected && new URL(res.url).pathname.startsWith("/erro/")) {
    throw new Error("Arquivo não encontrado");
  }
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  return {
    tamanho: Number(res.headers.get("Content-Length")),
    tipo: res.headers.get("Content-Type"),
    range: res.headers.get("Accept-Ranges") === "bytes",
  };
}
```

---

## 10. Exemplos por linguagem

### Node.js 18+

```javascript
// Node.js 18+ — upload em streaming (sem carregar o arquivo na memória)
import { createReadStream } from "node:fs";
import { stat } from "node:fs/promises";

const API = "https://storage.amheex.workers.dev";

async function enviar(caminho, destino) {
  const { size } = await stat(caminho);
  const res = await fetch(`${API}/X/${destino}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/octet-stream",
      "X-File-Size": String(size),
    },
    body: createReadStream(caminho),
    duplex: "half", // obrigatório para body em stream no fetch do Node
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.text()).trim();
}

console.log(await enviar("./video.mp4", "pasta/video.mp4"));
```

### Python

```python
# Python 3 + requests (pip install requests)
import os
import requests

API = "https://storage.amheex.workers.dev"

def enviar(caminho: str, destino: str) -> str:
    size = os.path.getsize(caminho)
    with open(caminho, "rb") as f:          # requests transmite o arquivo em stream
        r = requests.post(
            f"{API}/X/{destino}",
            data=f,
            headers={
                "Content-Type": "application/octet-stream",
                "X-File-Size": str(size),
            },
            timeout=(10, None),
        )
    r.raise_for_status()
    return r.text.strip()                   # URL pública pura

print(enviar("video.mp4", "pasta/video.mp4"))
```

### PHP

```php
<?php
// PHP 8.1+ (64-bit) com cURL — upload em streaming
$api     = "https://storage.amheex.workers.dev";
$arquivo = "video.mp4";
$destino = "pasta/video.mp4";
$size    = filesize($arquivo);

$fh = fopen($arquivo, "rb");
$ch = curl_init("$api/X/$destino");
curl_setopt_array($ch, [
    CURLOPT_CUSTOMREQUEST  => "POST",
    CURLOPT_UPLOAD         => true,      // envia o corpo direto do arquivo
    CURLOPT_INFILE         => $fh,
    CURLOPT_INFILESIZE     => $size,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => [
        "Content-Type: application/octet-stream",
        "X-File-Size: $size",
    ],
]);
$url  = trim(curl_exec($ch));
$code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);
fclose($fh);

if ($code !== 200) {
    throw new RuntimeException("Upload falhou: HTTP $code");
}
echo "Disponível em: $url\n";
```

---

<sub>AMHEEX Storage API v2 · Processado pelo AMHEEX SERVER · https://storage.amheex.workers.dev</sub>
