# Relatório Técnico: Integração com Omni Laravel Package

**Objetivo:** Consumir dados do serviço Omni utilizando a biblioteca oficial `econet-editora/omni` e validar a busca de alertas no endpoint de EcoLocação.

---

## 1. Infraestrutura e Dependências (Dockerfile)
Para permitir que o Composer instale pacotes privados via SSH dentro do container Alpine, foi necessário adicionar dependências de sistema.

```dockerfile
// ferramentas-econet-api/Dockerfile
RUN apk add ... git openssh-client && \
    docker-php-ext-install pdo_mysql mbstring zip gd exif pcntl bcmath
```
*   **Explicação:** O `git` e `openssh-client` são mandatórios para que o Composer realize o clone do repositório Bitbucket durante o build ou update.

---

## 2. Estrutura de Configuração (config/services.php)
A biblioteca `OmniClient` espera uma hierarquia específica de chaves para habilitar o serviço e definir timeouts/URLs.

```php
// ferramentas-econet-api/config/services.php
'econet' => [
    'omni' => [
        'url' => env('OMNI_API_URL', 'https://omni.econeteditora.com.br/api'),
        'enable' => env('OMNI_API_ENABLE', false),
        'timeout' => env('OMNI_API_TIMEOUT', 15),
        'mock' => env('OMNI_API_MOCK', false),
    ],
    ...
],
```
*   **Explicação:** Alinhamos as chaves do array com os nomes esperados internamente pelo `OmniClient` da biblioteca.

---

## 3. Implementação do Transport Layer (OmniHttpClient)
A biblioteca fornece os contratos (`HttpClientContract`), mas não a implementação. Criamos um Wrapper sobre o `Http` Facade do Laravel.

```php
// App\Services\Http\OmniHttpClient.php
public function request(string $method, string $url, array $options = []): array {
    $payload = $options['json'] ?? $options['query'] ?? [];
    $response = Http::withHeaders($headers)->timeout($timeout)->$method($url, $payload);
    
    $body = $response->json();
    // Normalização para o parser da biblioteca
    return [
        'status'  => $response->status(),
        'body'    => isset($body['data']) ? $body : ['data' => $body, 'status' => true],
        'headers' => $response->headers(),
    ];
}
```
*   **Explicação:** Esta classe resolve a comunicação HTTP. Adicionamos uma "casca" `['data' => ...]` no retorno para manter compatibilidade com o parser de resposta da biblioteca Omni.

---

## 4. Injeção de Dependência (AppServiceProvider)
Para que o `OmniClient` (instanciado automaticamente pelas Queries) funcione, o Laravel precisa saber como resolver a interface do cliente HTTP.

```php
// App\Providers\AppServiceProvider.php
public function register() {
    $this->app->bind(
        \EconetEditora\Omni\Contracts\HttpClientContract::class,
        \App\Services\Http\OmniHttpClient::class
    );
}
```
*   **Explicação:** Registramos o bind global para que qualquer classe da biblioteca Omni utilize nosso `OmniHttpClient` customizado.

---

## 5. Lógica de Negócio e Repositórios

Abaixo está o fluxo de código que consome a biblioteca Omni para gerar o retorno da API de EcoLocação.

### Service: EcolocacaoServices
O service coordena a montagem do array de resposta, chamando o repositório de alertas.

```php
// App\Services\Ecolocacao\EcolocacaoServices.php
public function carregarDadosSite(): array {
    return [
        'status' => true,
        'msg'    => 'dados carregados com sucesso',
        'data'   => [
            'uf'    => $this->listarUfs(),
            'alert' => $this->carregarAlertas(), // Chamada para o repositório
            ...
        ],
    ];
}
```

### Repository: AlertaRepository
O repositório encapsula a execução da Query do Omni.

```php
// App\Repositories\Ecolocacao\AlertaRepository.php
public function buscarPorCodigos(): array {
    return $this->alertQuery->handle()->toArray();
}
```

### Query: AlertQuery (Estendendo o pacote Omni)
Esta classe define as regras de busca (índice, filtros) e o mapeamento dos campos retornados pelo Elasticsearch do Omni.

```php
// App\Services\Ecolocacao\Queries\AlertQuery.php
class AlertQuery extends BaseOmniQuery {
    private const ALERT_CODES = ['mmeuo2n4d7l2', 'mmev8xcfgs75', 'mmevunbv1agn', 'mmevzd42o5wt'];

    protected function index(): string {
        return 'alerta';
    }

    protected function filters(array $context): array {
        return ['codigos' => self::ALERT_CODES];
    }

    protected function mapHits(array $hits): Collection {
        return collect($hits)->map(static function (array $hit) {
            $source = $hit['_source'] ?? $hit;
            return [
                'codigo'         => $source['codigo'] ?? null,
                'regra_exibicao' => $source['regra_exibicao'] ?? null,
                'texto'          => $source['texto'] ?? null,
                'tipo'           => $source['tipo_alerta']['tipo'] ?? null,
            ];
        })->filter(static fn($data) => $data['codigo'] !== null)->values();
    }
}
```

## 6. Validação de Resultados
*   **Endpoint:** `GET /api/ecolocacao/`
*   **Resultado:** Integração funcional. A biblioteca agora dispara um `POST /search` para a API de homologação do Omni e processa os alertas via `AlertQuery`.
*   **Mapeamento:** Os alertas são filtrados por código e retornados no formato: `codigo`, `regra_exibicao`, `texto` e `tipo`.

---
*Documentação técnica simplificada para revisão de código.*
