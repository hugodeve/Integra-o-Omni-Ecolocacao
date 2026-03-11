# Integração Omni — Ecolocacao

## Objetivo

Substituir os alertas hardcoded no `SiteService` do módulo `Ecolocacao` por dados dinâmicos
consumidos da API Omni (`https://omni-api-homol.econeteditora.com.br/api`), utilizando o
`OmniClient` já existente no projeto.

---

## Origem dos dados

Os alertas **não** vêm do package local `omni-laravel-package`. Eles são buscados diretamente
na API Omni de homologação via o cliente interno do projeto:

```
App\Services\Geral\Omni\OmniClient
    → config('services.econet.omniURL')
    → OMNI_URL_API_URL=https://omni-api-homol.econeteditora.com.br/api
```

O package `econet-editora/omni` (Bitbucket) foi adicionado ao `composer.json` para uso futuro,
mas a integração dos alertas usa o `OmniClient` interno por consistência com o padrão já
adotado pelo módulo Ecoclass.

---

## Arquivos modificados

### 1. `app/Modules/Ecolocacao/Services/SiteService.php`

**Antes:** alertas definidos como array estático hardcoded.

```php
'alert' => [
    [
        'codigo' => 'mmev8xcfgs75',
        'regra_exibicao' => 'Dados Iniciais',
        'texto' => '<p>Texto do Protótipo...</p>',
        'tipo' => 'Informativo',
    ],
    // ...
],
```

**Depois:** alertas carregados dinamicamente do Omni.

```php
<?php

namespace App\Modules\Ecolocacao\Services;

use App\Services\Geral\Omni\OmniClient;
use Illuminate\Support\Facades\Log;

class SiteService
{
    private const ALERT_CODES = [
        'mmeuo2n4d7l2',
        'mmev8xcfgs75',
        'mmevunbv1agn',
        'mmevzd42o5wt',
    ];

    private readonly OmniClient $omniClient;

    public function __construct(?OmniClient $omniClient = null)
    {
        $this->omniClient = $omniClient ?? new OmniClient();
    }

    private function carregarAlertas(): array
    {
        try {
            $response = $this->omniClient->search('alerta', [
                'codigos' => self::ALERT_CODES,
            ]);

            $hits = $response['hits'] ?? [];

            return array_map(fn($hit) => $hit['_source'] ?? $hit, $hits);
        } catch (\Throwable) {
            return [];
        }
    }

    public function carregarDadosSite(): array
    {
        return [
            // ...
            'data' => [
                // ...
                'alert' => $this->carregarAlertas(),
                // ...
            ],
        ];
    }
}
```

**Por que `?OmniClient $omniClient = null` e não injeção direta?**

O `OmniClient` tem um parâmetro `string` no construtor sem type hint completo. O container
do Laravel tentava resolver automaticamente e falhava com erro de binding. Usando `null` como
default, o container resolve o `SiteService` sem tentar injetar o `OmniClient` — a instância
é criada manualmente dentro do próprio service quando necessário. Isso também facilita os
testes unitários, que podem passar um mock diretamente.

---

### 2. `app/Services/Geral/Omni/OmniClient.php`

**Problema:** `Undefined array key "Authorization"` — o PHP em modo strict lança
`ErrorException` ao acessar uma chave inexistente em `$_SERVER` com `??` simples quando
a chave da esquerda existe mas a da direita não.

**Antes:**
```php
$headers = array_filter([
    'Authorization' => $_SERVER['HTTP_AUTHORIZATION'] ?? $_SERVER['Authorization'],
]);
```

**Depois:**
```php
$headers = array_filter([
    'Authorization' => $_SERVER['HTTP_AUTHORIZATION'] ?? $_SERVER['Authorization'] ?? null,
]);
```

O `?? null` no final garante que, se nenhuma das duas chaves existir em `$_SERVER`, o valor
seja `null` — que o `array_filter` remove automaticamente, sem lançar exceção.

---

### 3. `composer.json`

Adicionado o package `econet-editora/omni` e o repositório VCS do Bitbucket para instalação
via Composer.

```json
"require": {
    ...
    "econet-editora/omni": "*"
},
"repositories": {
    "omni": {
        "type": "vcs",
        "url": "git@bitbucket.org:econet-editora/omni-laravel-package.git"
    }
}
```

---

### 4. `docker-compose.yml`

Adicionado SSH agent forwarding para que o `composer install` dentro do container consiga
autenticar no Bitbucket via SSH ao clonar o repositório do package.

```yaml
volumes:
  - .:/app
  - ${SSH_AUTH_SOCK:-/dev/null}:/ssh-agent
environment:
  - SSH_AUTH_SOCK=/ssh-agent
```

---

## Erros encontrados e soluções

| Erro | Causa | Solução |
|---|---|---|
| `Permission denied` em `storage/framework/views` | Permissões incorretas no container Docker | `chmod -R 777 storage bootstrap/cache` |
| `Unresolvable dependency resolving` no container Laravel | Laravel tentava injetar `OmniClient` automaticamente, mas o construtor tem parâmetro `string` sem binding | Construtor com `?OmniClient $omniClient = null` e instanciação manual |
| `Undefined array key "Authorization"` | Acesso direto a `$_SERVER['Authorization']` sem fallback `null` | Adicionado `?? null` como terceiro fallback na cadeia |

---

## Fluxo final

```
GET /api/ecolocacao
    └── SiteController::index()
        └── SiteService::carregarDadosSite()
            └── SiteService::carregarAlertas()
                └── OmniClient::search('alerta', ['codigos' => [...]])
                    └── POST https://omni-api-homol.econeteditora.com.br/api/search
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

---

## Testes unitários

Criado `tests/Unit/Ecolocacao/SiteServiceTest.php` com três cenários:

```php
// Omni retorna dados → alert vem populado
public function test_carrega_alertas_do_omni(): void

// Omni lança exceção → alert vem vazio, endpoint não quebra
public function test_retorna_array_vazio_quando_omni_falha(): void

// Estrutura geral do retorno está íntegra
public function test_estrutura_basica_do_retorno(): void
```

Para executar:

```bash
docker compose exec api php vendor/bin/phpunit tests/Unit/Ecolocacao/SiteServiceTest.php
```
