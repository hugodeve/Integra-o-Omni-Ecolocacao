# Ecolocação - Integração com Omni

## Descrição

Módulo responsável por consumir dados do **Omni** (motor de busca interno da Econet) para o produto **Ecolocação**.
A integração utiliza o `OmniClient` já existente no projeto, seguindo o mesmo padrão arquitetural do módulo **Ecoclass**.

A principal funcionalidade implementada é a **busca de alertas** por códigos específicos, que são exibidos na tela inicial do Ecolocação.

---

## Arquitetura

```
GET /api/ecolocacao/

SiteController
  └── EcolocacaoServices
        └── AlertaRepository
              └── AlertQuery (extends BaseOmniQuery)
                    └── OmniClient
                          └── POST https://{OMNI_URL}/search
```

### Camadas

| Camada | Arquivo | Responsabilidade |
|--------|---------|------------------|
| **Rota** | `routes/api/ecolocacao.php` | Define o endpoint `GET /api/ecolocacao/` |
| **Controller** | `app/Http/Controllers/Ecolocacao/SiteController.php` | Recebe a requisição e delega ao service |
| **Service** | `app/Services/Ecolocacao/EcolocacaoServices.php` | Orquestra os dados do site (UFs, alertas, taxas, imóvel) |
| **Repository** | `app/Repositories/Ecolocacao/AlertaRepository.php` | Encapsula o acesso aos alertas via Query |
| **Query** | `app/Services/Ecolocacao/Queries/AlertQuery.php` | Define index, filtros e mapeamento dos hits do Omni |
| **Base Query** | `app/Services/Ecoclass/Queries/BaseOmniQuery.php` | Classe abstrata compartilhada que executa a busca no Omni |
| **Client HTTP** | `app/Services/Geral/Omni/OmniClient.php` | Realiza o POST HTTP ao endpoint `/search` do Omni |

---

## Endpoint

### `GET /api/ecolocacao/`

Retorna os dados iniciais necessários para carregar a tela do Ecolocação.

#### Resposta de sucesso (200)

```json
{
  "status": true,
  "msg": "dados carregados com sucesso",
  "data": {
    "uf": [
      { "sigla": "AC", "nome": "Acre" },
      { "sigla": "AL", "nome": "Alagoas" }
    ],
    "alert": [
      {
        "codigo": "mmeuo2n4d7l2",
        "regra_exibicao": "Dados Iniciais",
        "texto": "Texto do alerta...",
        "tipo": "Informativo"
      }
    ],
    "general_rate": {
      "ibs": [{ "aliquota": 1.00 }, { "aliquota": 3.65 }],
      "cbs": [{ "aliquota": 1.00 }, { "aliquota": 3.65 }]
    },
    "imovel": {
      "selecao": { ... },
      "finalidade": { ... }
    }
  }
}
```

#### Resposta de erro (500)

```json
{
  "status": false,
  "msg": "Mensagem de erro"
}
```

> Se o Omni estiver indisponível, o campo `alert` retorna `[]` sem afetar o restante da resposta.

---

## Integração com Omni - Busca de Alertas

### Request enviado ao Omni

```
POST {OMNI_URL}/search
```

```json
{
  "index": "alerta",
  "filters": {
    "codigos": [
      "mmeuo2n4d7l2",
      "mmev8xcfgs75",
      "mmevunbv1agn",
      "mmevzd42o5wt"
    ]
  }
}
```

### Fluxo interno

1. `AlertQuery::index()` retorna `"alerta"` — define o índice de busca no Omni.
2. `AlertQuery::filters()` retorna os 4 códigos de alerta específicos do Ecolocação.
3. `BaseOmniQuery::handle()` chama `OmniClient::search()` com o index e os filtros.
4. `OmniClient` faz o POST HTTP para `{OMNI_URL}/search` com o header `Authorization` da requisição original.
5. A resposta do Omni contém um array `hits` com os documentos encontrados.
6. `AlertQuery::mapHits()` extrai de cada hit os campos relevantes:
   - `codigo` — identificador único do alerta
   - `regra_exibicao` — onde o alerta deve ser exibido
   - `texto` — conteúdo do alerta
   - `tipo` — tipo do alerta (extraído de `tipo_alerta.tipo`)
7. Hits sem `codigo` são filtrados automaticamente.

### Configuração da URL do Omni

Definida na variável de ambiente `OMNI_URL_API_URL`:

| Ambiente | URL |
|----------|-----|
| Desenvolvimento | `https://omni-api-homol.econeteditora.com.br/api` |
| Homologação | Configurada via Jenkins |
| Produção | `https://omni.econeteditora.com.br/api` |

Referência em `config/services.php`:

```php
'econet' => [
    'omniURL' => env('OMNI_URL_API_URL', 'https://omni.econeteditora.com.br/api'),
],
```

---

## Arquivos criados/modificados

| Arquivo | Ação |
|---------|------|
| `app/Services/Ecolocacao/EcolocacaoServices.php` | Criado |
| `app/Services/Ecolocacao/Queries/AlertQuery.php` | Criado |
| `app/Repositories/Ecolocacao/AlertaRepository.php` | Criado |
| `app/Http/Controllers/Ecolocacao/SiteController.php` | Modificado (usa EcolocacaoServices) |
| `tests/Unit/Ecolocacao/SiteServiceTest.php` | Modificado (usa nova estrutura) |
| `app/Modules/Ecolocacao/` | Removido (migrado para Services/Repositories) |

---

## Testes

Arquivo: `tests/Unit/Ecolocacao/SiteServiceTest.php`

| Teste | O que valida |
|-------|-------------|
| `test_carrega_alertas_do_omni` | Verifica que os alertas retornados pelo Omni são incluídos corretamente na resposta |
| `test_retorna_array_vazio_quando_omni_falha` | Garante que se o Omni falhar, `alert` retorna `[]` sem quebrar o endpoint |
| `test_estrutura_basica_do_retorno` | Valida que a resposta contém as chaves `uf`, `alert`, `general_rate`, `imovel` e 27 UFs |

Executar:

```bash
php artisan test --filter=SiteServiceTest
```
