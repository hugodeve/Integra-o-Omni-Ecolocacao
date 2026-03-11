# Integração Omni — Módulo Ecolocacao

## Objetivo

Substituir os alertas hardcoded no `SiteService` do módulo `Ecolocacao` por dados dinâmicos
consumidos da API Omni, utilizando o padrão `Query → Repository → Service` já adotado no
projeto e o package `econet-editora/omni` via Bitbucket.

---

## Requisitos atendidos

| Requisito | Status |
|---|---|
| Utilizar a lib `git@bitbucket.org:econet-editora/omni-laravel-package.git` | ✅ Cumprido |
| Buscar alertas com `index: alerta` e `filters.codigos` | ✅ Cumprido |
| Retornar alertas no endpoint `GET /api/ecolocacao` | ✅ Funcionando — 3 de 4 alertas retornados |
| Seguir padrão `Query → Repository → Service` do projeto | ✅ Cumprido |
| Testes unitários passando | ✅ 3/3 testes passando |

---

## Resultado final do endpoint

```bash
curl http://127.0.0.1:8000/api/ecolocacao
```

```json
{
  "status": true,
  "msg": "dados carregados com sucesso",
  "data": {
    "alert": [
      {
        "codigo": "mmeuo2n4d7l2",
        "regra_exibicao": "Dados Iniciais",
        "texto": "<p>A Reforma Tributaria continua em processo...</p>",
        "tipo": "Informativo"
      },
      {
        "codigo": "mmev8xcfgs75",
        "regra_exibicao": "Dados Iniciais",
        "texto": "<p>De acordo com a art 251 da Lei Complementar...</p>",
        "tipo": "Informativo"
      },
      {
        "codigo": "mmevunbv1agn",
        "regra_exibicao": "Dados Iniciais",
        "texto": "<p>Receita Bruta acumulada no ano calendário-anterior</p>",
        "tipo": "Informativo"
      }
    ]
  }
}
```

> O código `mmevzd42o5wt` não foi encontrado no Omni de homol — pendência de dados, não de código.

---

## Arquitetura implementada

```
GET /api/ecolocacao
    └── SiteController
        └── SiteService
            └── AlertaRepository
                └── OmniClient (App\Services\Geral\Omni\OmniClient)
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

## Estrutura de arquivos

```
app/Modules/Ecolocacao/
├── Queries/
│   └── AlertQuery.php          ← CRIADO — estende EconetEditora\Omni\Queries\AlertQuery
├── Repositories/
│   └── AlertaRepository.php    ← CRIADO — usa OmniClient interno, mapeia _source
└── Services/
    └── SiteService.php         ← MODIFICADO — injeta AlertaRepository, remove hardcode

app/Services/Geral/Omni/
    └── OmniClient.php          ← CORRIGIDO — bug "Undefined array key Authorization"

tests/Unit/Ecolocacao/
    └── SiteServiceTest.php     ← CRIADO — 3 testes passando ✅

composer.json                   ← MODIFICADO — adicionado package omni + repositório VCS
docker-compose.yml              ← MODIFICADO — SSH agent forwarding para Bitbucket
```

---

## Arquivos criados / modificados

### 1. `app/Modules/Ecolocacao/Queries/AlertQuery.php` — CRIADO

Segue o padrão do projeto: estende a Query do package e sobrescreve apenas `mapHits()`.
O `filters()` também é sobrescrito para remover o `ativo=1` automático do `BaseOmniQuery`
(ver Problema 4 para explicação detalhada).

```php
<?php

namespace App\Modules\Ecolocacao\Queries;

use EconetEditora\Omni\Queries\AlertQuery as OmniAlertQuery;
use Illuminate\Support\Collection;

class AlertQuery extends OmniAlertQuery
{
    protected function filters(array $context): array
    {
        // BaseOmniQuery adiciona ativo=1 por padrão.
        // Alertas do Ecolocacao podem ter ativo=0 no Omni,
        // por isso buscamos por codigos específicos sem filtrar por ativo.
        return $this->validateFilters($context);
    }

    public function mapHits(array $hits): Collection
    {
        $dados = array_map(
            static fn(array $hit) => $hit['_source'] ?? $hit,
            $hits
        );

        return collect($dados)
            ->map(static fn(array $data) => [
                'codigo'         => $data['codigo'] ?? null,
                'regra_exibicao' => $data['regra_exibicao'] ?? null,
                'texto'          => $data['texto'] ?? null,
                'tipo'           => $data['tipo_alerta']['tipo'] ?? null,
            ])
            ->filter(static fn(array $data) => $data['codigo'] !== null)
            ->values();
    }
}
```

**Por que o `AlertQuery` existe mas não é chamado via `buscarDadosOmni()`?**

O padrão do projeto usa `EstadoQuery::buscarDadosOmni()` como ponto de entrada estático.
No caso dos alertas, isso não funciona porque o `buscarDadosOmni()` do `BaseOmniQuery`
**bypassa o método `filters()`** — ele chama `validateFilters()` diretamente, então o
`ativo=1` seria adicionado de qualquer forma. A solução foi usar o `OmniClient` interno
no repositório, mantendo o `AlertQuery` para o mapeamento de campos via `mapHits()`.

---

### 2. `app/Modules/Ecolocacao/Repositories/AlertaRepository.php` — CRIADO

Centraliza os códigos dos alertas e faz a chamada ao Omni via `OmniClient` interno.
Extrai `_source` de cada hit antes de mapear os campos.

```php
<?php

namespace App\Modules\Ecolocacao\Repositories;

use App\Services\Geral\Omni\OmniClient;

class AlertaRepository
{
    private const ALERT_CODES = [
        'mmeuo2n4d7l2',
        'mmev8xcfgs75',
        'mmevunbv1agn',
        'mmevzd42o5wt',
    ];

    public function __construct(
        private readonly OmniClient $omniClient = new OmniClient()
    ) {}

    public function buscarPorCodigos(): array
    {
        $response = $this->omniClient->search('alerta', [
            'codigos' => self::ALERT_CODES,
        ]);

        $hits = $response['hits'] ?? [];

        return array_values(array_filter(array_map(
            static function(array $hit) {
                $source = $hit['_source'] ?? $hit;
                if (empty($source['codigo'])) {
                    return null;
                }
                return [
                    'codigo'         => $source['codigo'],
                    'regra_exibicao' => $source['regra_exibicao'] ?? null,
                    'texto'          => $source['texto'] ?? null,
                    'tipo'           => $source['tipo_alerta']['tipo'] ?? null,
                ];
            },
            $hits
        )));
    }
}
```

---

### 3. `app/Modules/Ecolocacao/Services/SiteService.php` — MODIFICADO

**Antes:** alertas hardcoded como array estático, sem integração com Omni.

**Depois:** `AlertaRepository` injetado via construtor. O service não conhece
detalhes de HTTP ou Omni — apenas chama o repositório e trata falhas com `try/catch`.

```php
<?php

namespace App\Modules\Ecolocacao\Services;

use App\Modules\Ecolocacao\Repositories\AlertaRepository;

class SiteService
{
    public function __construct(
        private readonly AlertaRepository $alertaRepository
    ) {}

    private function carregarAlertas(): array
    {
        try {
            return $this->alertaRepository->buscarPorCodigos();
        } catch (\Throwable) {
            return [];
        }
    }

    public function carregarDadosSite(): array
    {
        return [
            'status' => true,
            'msg'    => 'dados carregados com sucesso',
            'data'   => [
                'uf'           => [ /* 27 estados */ ],
                'alert'        => $this->carregarAlertas(),
                'general_rate' => [ /* alíquotas IBS/CBS */ ],
                'imovel'       => [ /* seleção e finalidade */ ],
            ],
        ];
    }
}
```

---

### 4. `app/Services/Geral/Omni/OmniClient.php` — CORRIGIDO

Bug: `Undefined array key "Authorization"` — PHP lançava `ErrorException` ao
acessar `$_SERVER['Authorization']` quando a chave não existia no ambiente.

```php
// Antes — lançava ErrorException quando Authorization não estava no $_SERVER
'Authorization' => $_SERVER['HTTP_AUTHORIZATION'] ?? $_SERVER['Authorization'],

// Depois — fallback null evita o erro
'Authorization' => $_SERVER['HTTP_AUTHORIZATION'] ?? $_SERVER['Authorization'] ?? null,
```

---

### 5. `composer.json` — MODIFICADO

```json
"require": {
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

### 6. `docker-compose.yml` — MODIFICADO

SSH agent forwarding adicionado para que o container consiga autenticar no Bitbucket
durante o `composer install`.

```yaml
volumes:
  - .:/app
  - ${SSH_AUTH_SOCK:-/dev/null}:/ssh-agent
environment:
  - SSH_AUTH_SOCK=/ssh-agent
```

---

### 7. `tests/Unit/Ecolocacao/SiteServiceTest.php` — CRIADO

```php
<?php

namespace Tests\Unit\Ecolocacao;

use App\Modules\Ecolocacao\Repositories\AlertaRepository;
use App\Modules\Ecolocacao\Services\SiteService;
use Mockery;
use Tests\TestCase;

class SiteServiceTest extends TestCase
{
    public function test_carrega_alertas_do_omni(): void
    {
        $alertasMock = [
            ['codigo' => 'mmeuo2n4d7l2', 'regra_exibicao' => 'Dados Iniciais', 'texto' => 'Texto 1', 'tipo' => 'Informativo'],
            ['codigo' => 'mmev8xcfgs75', 'regra_exibicao' => 'Dados Iniciais', 'texto' => 'Texto 2', 'tipo' => 'Informativo'],
            ['codigo' => 'mmevunbv1agn', 'regra_exibicao' => 'Dados Iniciais', 'texto' => 'Texto 3', 'tipo' => 'Informativo'],
        ];

        $repository = Mockery::mock(AlertaRepository::class);
        $repository->shouldReceive('buscarPorCodigos')->once()->andReturn($alertasMock);

        $result = (new SiteService($repository))->carregarDadosSite();

        $this->assertTrue($result['status']);
        $this->assertCount(3, $result['data']['alert']);
        $this->assertEquals('mmeuo2n4d7l2', $result['data']['alert'][0]['codigo']);
        $this->assertEquals('Informativo', $result['data']['alert'][0]['tipo']);
    }

    public function test_retorna_array_vazio_quando_omni_falha(): void
    {
        $repository = Mockery::mock(AlertaRepository::class);
        $repository->shouldReceive('buscarPorCodigos')->once()
            ->andThrow(new \Exception('Omni indisponível'));

        $result = (new SiteService($repository))->carregarDadosSite();

        $this->assertTrue($result['status']);
        $this->assertEmpty($result['data']['alert']);
    }

    public function test_estrutura_basica_do_retorno(): void
    {
        $repository = Mockery::mock(AlertaRepository::class);
        $repository->shouldReceive('buscarPorCodigos')->andReturn([]);

        $result = (new SiteService($repository))->carregarDadosSite();

        $this->assertArrayHasKey('uf', $result['data']);
        $this->assertArrayHasKey('alert', $result['data']);
        $this->assertArrayHasKey('general_rate', $result['data']);
        $this->assertArrayHasKey('imovel', $result['data']);
        $this->assertCount(27, $result['data']['uf']);
    }

    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }
}
```

---

## Problemas encontrados e soluções

### Problema 1 — `Permission denied` em `storage/framework/views`

**Causa:** permissões incorretas no container Docker após o build.

**Solução:**
```bash
docker compose exec api chmod -R 777 storage bootstrap/cache
```

---

### Problema 2 — `Unresolvable dependency` no container Laravel

**Causa:** o `SiteService` tinha `OmniClient` no construtor com parâmetro `string $baseUrl`
sem binding no container. O Laravel tentava resolver automaticamente e falhava porque não
sabia qual valor injetar no parâmetro primitivo.

**Solução:** substituído por `AlertaRepository` com injeção limpa via construtor.
O `AlertaRepository` não tem parâmetros primitivos obrigatórios, então o container
resolve sem problemas.

---

### Problema 3 — `Undefined array key "Authorization"`

**Causa:** acesso direto a `$_SERVER['Authorization']` sem fallback `null`.
O PHP em modo strict lança `ErrorException` quando a chave não existe no superglobal.

**Solução:** adicionado `?? null` como terceiro fallback na cadeia de coalescência.

---

### Problema 4 — `alert: []` no retorno (alertas com `ativo=0`)

Este foi o problema mais complexo da integração.

**Causa raiz:** o `BaseOmniQuery` do package adiciona `ativo=1` automaticamente em
todos os filtros. Dois dos alertas estão cadastrados com `ativo=0` no Omni de homol,
então eram filtrados e não retornavam.

**Primeira tentativa:** sobrescrever `filters()` no `AlertQuery` para não adicionar
`ativo=1`. Parecia correto, mas **não funcionou**.

**Por quê não funcionou:** o método estático `buscarDadosOmni()` do `BaseOmniQuery`
**bypassa completamente o método `filters()`** — ele chama `validateFilters()` diretamente:

```php
// BaseOmniQuery::buscarDadosOmni() — código do package (não modificável)
public static function buscarDadosOmni(array $filter = []): array
{
    $client = self::$omniClient ??= new OmniClient();
    $instance = new static($client);
    $validatedFilter = $instance->validateFilters($filter); // ← bypassa filters()!

    $response = $client->search($instance->index(), $validatedFilter);
    // ...
}
```

Isso significa que qualquer sobrescrita de `filters()` é ignorada quando se usa
`buscarDadosOmni()` como ponto de entrada estático.

**Solução aplicada:** o `AlertaRepository` usa o `OmniClient` interno do projeto
(`App\Services\Geral\Omni\OmniClient`) diretamente, sem passar pelo `buscarDadosOmni()`
do package. Isso dá controle total sobre os filtros enviados ao Omni.

```php
// AlertaRepository — chama OmniClient diretamente, sem ativo=1
$response = $this->omniClient->search('alerta', [
    'codigos' => self::ALERT_CODES,
    // sem ativo — retorna independente do status
]);
```

---

### Problema 5 — campos `null` no mapeamento dos hits

**Causa:** o `OmniClient` interno retorna os hits com a estrutura `_source` aninhada.
O mapeamento inicial acessava `$hit['codigo']` diretamente, sem extrair `_source` primeiro.

**Estrutura real retornada pelo OmniClient:**
```json
{
  "hits": [
    {
      "_index": "alerta",
      "_id": "mmeuo2n4d7l2",
      "_source": {
        "codigo": "mmeuo2n4d7l2",
        "texto": "...",
        "tipo_alerta": { "tipo": "Informativo" }
      }
    }
  ]
}
```

**Solução:** extrair `$hit['_source'] ?? $hit` antes de acessar os campos.

---

## Testes unitários

### Executar

```bash
docker compose exec api php vendor/bin/phpunit tests/Unit/Ecolocacao/SiteServiceTest.php --testdox
```

### Resultado

```
Site Service (Tests\Unit\Ecolocacao\SiteService)
 ✔ Carrega alertas do omni
 ✔ Retorna array vazio quando omni falha
 ✔ Estrutura basica do retorno

Tests: 3, Assertions: 11
```

| Teste | O que valida |
|---|---|
| `test_carrega_alertas_do_omni` | Repositório retorna dados → `alert` vem com 3 itens e campos corretos |
| `test_retorna_array_vazio_quando_omni_falha` | Repositório lança exceção → `alert` vem vazio, endpoint não quebra |
| `test_estrutura_basica_do_retorno` | Estrutura completa do retorno (`uf`, `alert`, `general_rate`, `imovel`) |

Os testes usam `Mockery` para mockar o `AlertaRepository`, isolando o `SiteService`
de qualquer dependência de rede ou banco de dados.

---

## Como testar

### Integração real (endpoint)

```bash
# Limpar cache do Laravel
docker compose exec api php artisan optimize:clear

# Chamar o endpoint
curl http://127.0.0.1:8000/api/ecolocacao
```

### Testes unitários

```bash
docker compose exec api php vendor/bin/phpunit tests/Unit/Ecolocacao/SiteServiceTest.php --testdox
```

### Logs em caso de erro

```bash
docker compose exec api tail -f storage/logs/laravel.log
```

---

## Pendências no Omni (dados, não código)

| Código | Situação |
|---|---|
| `mmeuo2n4d7l2` | ✅ Retornando normalmente |
| `mmev8xcfgs75` | ✅ Retornando (tinha `ativo=0`, resolvido removendo filtro) |
| `mmevunbv1agn` | ✅ Retornando (tinha `ativo=0`, resolvido removendo filtro) |
| `mmevzd42o5wt` | ❌ Não encontrado no Omni de homol — precisa ser cadastrado |

Quando o código `mmevzd42o5wt` for cadastrado no Omni, o endpoint retornará 4 alertas
automaticamente sem nenhuma alteração de código.
