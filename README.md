# AjudaJF 🤝

**Plataforma de solidariedade para famílias atingidas pelas chuvas em Juiz de Fora, MG.**

Conecta quem precisa de ajuda com voluntários e pontos de doação, usando apenas Google Sheets como banco de dados e Google Apps Script como API. Sem servidor, sem custos de infraestrutura.

---

## Visão geral

O AjudaJF é composto por duas páginas independentes e um backend serverless:

| Arquivo | Descrição |
|---|---|
| `ajudajf-v2.html` | Página principal — pedidos de ajuda, mapa e pontos de doação |
| `voluntarios.html` | Página de voluntários — cadastro, mapa e lista de quem pode ajudar |
| `Code.gs` | Backend — Google Apps Script que lê e escreve na planilha |

---

## Como funciona

```
Usuário preenche form → HTML faz POST via iframe oculto → Apps Script salva na planilha
                     → HTML faz GET com JSONP          → Apps Script retorna dados
                                                        → Mapa e lista são renderizados
```

Toda a comunicação com a API usa **JSONP** (requisições GET via `<script>` tag), que contorna restrições de CORS sem necessidade de configuração adicional no servidor. Formulários são enviados via `target="hidden_iframe"`, também sem CORS.

---

## Estrutura dos arquivos

```
ajudajf/
├── ajudajf-v2.html     ← Página principal
├── voluntarios.html    ← Página de voluntários
├── Code.gs             ← Backend (Google Apps Script)
└── README.md
```

---

## Backend — Google Apps Script

### Configuração inicial

**1. Planilha Google Sheets**

Crie uma planilha e anote o ID (parte da URL entre `/d/` e `/edit`). Crie três abas com os seguintes cabeçalhos exatos na linha 1:

**Aba `requests`**
```
id | created_at | updated_at | name | phone | needs | details | delivery_needed | neighborhood | address | lat | lng | location_precision | status
```

**Aba `donation_points`**
```
id | created_at | updated_at | name | address | neighborhood | lat | lng | items | hours | contact | notes | status
```

**Aba `volunteers`**
```
id | created_at | updated_at | name | phone | skills | availability | neighborhood | address | lat | lng | location_precision | notes | status
```

**2. Apps Script**

- Abra a planilha → menu **Extensões → Apps Script**
- Cole o conteúdo do `Code.gs`
- Substitua o valor de `SPREADSHEET_ID` pelo ID da sua planilha
- Troque `ADMIN_CODE` por um código forte (usado para atualizações administrativas)

**3. Publicar como Web App**

- Clique em **Implantar → Nova implantação**
- Tipo: `Web app`
- Executar como: `Eu (seu e-mail)`
- Quem pode acessar: `Qualquer pessoa` ← **obrigatório**
- Copie a URL gerada

> ⚠️ Toda vez que alterar o `Code.gs`, é necessário criar **nova versão** em "Gerenciar implantações". Simplesmente salvar o arquivo não atualiza o Web App publicado.

**4. Configurar os HTMLs**

Nos dois arquivos HTML, substitua a constante:
```js
const WEB_APP_URL = "WEB_APP_URL_AQUI";
// →
const WEB_APP_URL = "https://script.google.com/macros/s/SEU_ID/exec";
```

---

## API — Endpoints disponíveis

Todos os endpoints GET retornam JSONP quando o parâmetro `callback` é informado.

### GET — Leitura de dados

| Ação | Descrição | Resposta |
|---|---|---|
| `?action=listRequests` | Lista pedidos de ajuda | `{ ok, data: [...] }` |
| `?action=listPoints` | Lista pontos de doação ativos | `{ ok, data: [...] }` |
| `?action=listVolunteers` | Lista voluntários ativos | `{ ok, data: [...] }` |
| `?action=publicUpdateRequest&id=X&status=done` | Marca pedido como atendido | `{ ok }` |
| `?action=ping` | Diagnóstico — verifica se o deploy está ativo | `{ ok, msg:"pong", ts }` |

### POST — Escrita de dados

| Ação | Descrição |
|---|---|
| `action=createRequest` | Cadastra pedido de ajuda |
| `action=createPoint` | Cadastra ponto de doação |
| `action=createVolunteer` | Cadastra voluntário |
| `action=adminUpdate` | Atualiza status (requer `admin_code`) |

---

## Página principal — `ajudajf-v2.html`

### Seções

**Check-in / Pedido de ajuda**
- Nome (opcional), telefone com WhatsApp (obrigatório), necessidades (chips de seleção múltipla), detalhes, se precisa de entrega, bairro e endereço completo
- Opção de preencher o endereço automaticamente via GPS do dispositivo (geocodificação reversa via Nominatim)
- O endereço é convertido em coordenadas (lat/lng) via Nominatim antes do envio

**Mapa interativo**
- Marcadores laranja = pedidos abertos, verde = pedidos atendidos, azul = pontos de doação
- Filtros: somente abertos, mostrar/ocultar pontos de doação, filtro por tipo de necessidade
- Legenda visual integrada ao mapa
- Toggle lista/mapa para mobile

**Cards de pedidos**
Cada card exibe: nome do solicitante, endereço, bairro, chips das necessidades (com labels completos), badge "Precisa de entrega", caixa de detalhes, botões de ação (WhatsApp, copiar telefone, copiar endereço, rota, marcar atendido)

**Pontos de doação**
- Grade responsiva de cards com nome do local, endereço, itens aceitos, horários, contato, observações
- Formulário de cadastro de novos pontos com geocodificação automática

**Como ajudar**
- Orientações sobre prioridades urgentes e boas práticas para doações

### Necessidades disponíveis

| Chave | Label |
|---|---|
| `agua` | 💧 Água |
| `alimentos` | 🥫 Alimentos |
| `pronto` | 🍱 Alimento pronto |
| `roupas` | 👕 Roupas |
| `cobertor` | 🛏 Cobertores |
| `higiene` | 🧴 Higiene |
| `fraldas` | 👶 Fraldas |
| `remedios` | 💊 Remédios |
| `pets` | 🐾 Pets |

---

## Página de voluntários — `voluntarios.html`

### Seções

**Cadastro de voluntário**
- Nome completo (obrigatório), telefone/WhatsApp (obrigatório)
- Tipos de ajuda que pode oferecer (chips de seleção múltipla)
- Disponibilidade: agora / somente hoje / esta semana / finais de semana
- Observações livres, bairro, endereço com opção de GPS

**Mapa de voluntários**
- Marcadores verdes com popup contendo nome, habilidades e link para WhatsApp
- Filtro por tipo de ajuda
- Toggle lista/mapa para mobile

**Cards de voluntários**
Cada card exibe: nome, localização, chips do que pode ajudar, badge de disponibilidade, observações, botões de ação (WhatsApp, copiar telefone, ver no mapa)

### Tipos de ajuda (voluntários)

| Chave | Label |
|---|---|
| `transporte` | 🚗 Transporte |
| `alimentos` | 🥫 Doação de alimentos |
| `agua` | 💧 Doação de água |
| `roupas` | 👕 Doação de roupas |
| `higiene` | 🧴 Higiene / Limpeza |
| `abrigo` | 🏠 Abrigo temporário |
| `saude` | 🏥 Saúde / Primeiros socorros |
| `psicologico` | 🤝 Apoio emocional |
| `construcao` | 🔧 Construção / Reparo |
| `pets` | 🐾 Acolhimento de pets |
| `financeiro` | 💰 Ajuda financeira |
| `outros` | ➕ Outros |

---

## Funcionalidades técnicas

### Geocodificação
Endereços são convertidos em coordenadas automaticamente via **Nominatim (OpenStreetMap)**, sem custos ou chave de API. O resultado é cacheado no `localStorage` para evitar requisições repetidas ao mesmo endereço.

Quando o usuário marca "Usar minha localização atual", o browser solicita permissão de GPS e faz **geocodificação reversa** (coordenadas → endereço legível) também via Nominatim, preenchendo o campo automaticamente.

### Anti-spam
Todos os formulários possuem um campo honeypot oculto (`name="website"`). Se estiver preenchido, o Apps Script ignora o envio silenciosamente, sem retornar erro ao bot.

Além disso, o `safeText_()` no backend bloqueia entradas contendo URLs (`http://`, `www.`), que são o principal vetor de spam em formulários abertos.

### Auto-refresh
Ambas as páginas atualizam os dados automaticamente a cada **5 minutos**, mas somente quando a aba do navegador estiver visível (`document.visibilityState === "visible"`). A preferência de auto-carregamento é salva no `localStorage`.

### Marcar como atendido
O botão "Marcar atendido" usa **JSONP (GET)** em vez de POST, evitando o erro 403 que o Apps Script retorna para POSTs cross-origin não autenticados. O status no frontend é atualizado imediatamente (optimistic UI) e revertido caso o servidor retorne erro.

O Apps Script recebe `status=done` (enviado pelo frontend) e grava `closed` na planilha. Ao ler os dados de volta, qualquer valor diferente de `open` é exibido como "ATENDIDO".

### Segurança de dados
- Telefones são exibidos publicamente (necessário para contato)
- Nomes são opcionais nos pedidos de ajuda
- Coordenadas são arredondadas para 3 casas decimais (~111m de precisão) para preservar privacidade de localização
- Nenhum dado sensível além de telefone e endereço é coletado

---

## Dependências externas

Todas carregadas via CDN, sem instalação:

| Biblioteca | Versão | Uso |
|---|---|---|
| [Leaflet.js](https://leafletjs.com) | 1.9.4 | Mapas interativos |
| [OpenStreetMap](https://www.openstreetmap.org) | — | Tiles do mapa |
| [Nominatim](https://nominatim.org) | — | Geocodificação e geocodificação reversa |
| [Google Fonts](https://fonts.google.com) | — | Lora (serif) + DM Sans |

---

## Deploy

O projeto é **100% estático** — basta hospedar os dois arquivos HTML em qualquer lugar:

- **GitHub Pages** — gratuito, basta subir os arquivos no repositório e ativar Pages
- **Netlify / Vercel** — arrastar a pasta no painel, deploy instantâneo
- **Google Drive** — publicar como página web (menos recomendado)
- **Servidor próprio** — qualquer hospedagem com suporte a HTML estático

Os arquivos precisam estar **na mesma pasta** para que os links entre eles funcionem corretamente (`ajudajf-v2.html` ↔ `voluntarios.html`).

---

## Diagnóstico de problemas

**Dados não carregam**
Abra no navegador: `SUA_URL/exec?action=ping&callback=test`
- Retorna `test({"ok":true,...})` → deploy OK
- Redireciona para login Google → "Quem pode acessar" está como "Somente eu", não "Qualquer pessoa"
- Retorna `{"ok":false,"error":"Ação inválida"}` → código novo não foi publicado, criar nova versão

**Erro 403 ao enviar formulário**
Confirmar que o Web App está publicado com "Quem pode acessar: Qualquer pessoa".

**"Marcar atendido" não funciona**
Testar diretamente: `SUA_URL/exec?action=publicUpdateRequest&id=ID_REAL&status=done&callback=test`
- Deve retornar `test({"ok":true})`
- Se retornar `"ID não encontrado"` → o ID no frontend não bate com a planilha

**Endereço não encontrado na geocodificação**
Incluir número e bairro no endereço. O Nominatim tem melhor desempenho com endereços completos. Endereços muito genéricos (ex: "Centro") podem não ser encontrados ou retornar coordenadas imprecisas.

---

## Créditos

Desenvolvido por [Jonathan Coelho](https://www.linkedin.com/in/jonathan-coelho-06a91014b/) para apoio às famílias atingidas pelas chuvas em Juiz de Fora, MG.

Use com responsabilidade. Em caso de risco imediato à vida, acione a **Defesa Civil (199)** ou os **Bombeiros (193)**.
