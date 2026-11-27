# Villa Prime — Sistema de Gestão de Buffet (PHP/Laravel)

Documentação do projeto final de desenvolvimento web da **Villa Prime**, empresa de buffet para eventos.

## 1. Visão geral

A Villa Prime atende clientes que contratam serviços de buffet. O sistema centraliza:

- o cadastro e a manutenção dos **contratantes** (clientes que contratam os serviços);
- a gestão do catálogo de **produtos** e **serviços** oferecidos;
- a consulta ao catálogo com **filtro por categoria** e **ordenação por preço**;
- os **agendamentos**, que ligam o contratante aos itens contratados, com data, horário, status e cálculo do valor total.

O projeto é escrito em **PHP**, utilizando o framework **Laravel** e o padrão MVC. Toda a entrada de dados é validada por classes **FormRequest**, e a aplicação adota proteção contra **CSRF**, **XSS** e **SQL injection**.

> **Status:** este repositório encontra-se em fase inicial. As seções abaixo descrevem a especificação e a estrutura de código previstas para a implementação.

## 2. Equipe

| Integrante | Função |
| --- | --- |
| Victor | Desenvolvimento |
| Mayne | Desenvolvimento |

**Empresa:** Villa Prime  
**Domínio:** buffet / eventos

## 3. Convenções PHP/Laravel adotadas

| Conceito do sistema | Termo no código |
| --- | --- |
| Contratante | `App\Models\Contratante` — tabela `contratantes` |
| Produto | `App\Models\Produto` — tabela `produtos` |
| Serviço | `App\Models\Servico` — tabela `servicos` |
| Agendamento | `App\Models\Agendamento` — tabela `agendamentos` |
| Categoria | `App\Models\Categoria` — tabela `categorias` |
| CRUD | Métodos resource do controller: `index`, `create`, `store`, `show`, `edit`, `update`, `destroy` |
| Validação | `App\Http\Requests\*Request extends FormRequest` com `authorize()` e `rules()` |
| Regras de negócio customizadas | `App\Rules\*` (classes que implementam `ValidationRule`) |
| Status | `App\Enums\StatusAgendamento` (enum PHP 8.1) |
| Relacionamentos | `belongsTo()` e `belongsToMany()` do Eloquent |
| Valor total | `calcularValorTotal()` no model + accessor `getValorTotalAttribute()` |
| Banco de dados | Migrations em `database/migrations` com `Schema` / `Blueprint` |
| Interface | Views Blade em `resources/views/**/*.blade.php` |
| Rotas | `routes/web.php` com `Route::resource()` |

Os nomes de campos e tabelas seguem `snake_case` (por exemplo, `numero_convidados`, `data_evento`, `preco_unitario`).

## 4. Model Contratante — entidade principal

O **Contratante** representa a pessoa ou empresa que contrata os serviços da Villa Prime.

### 4.1 Estrutura da tabela `contratantes`

| Coluna | Tipo PHP/DB | Descrição |
| --- | --- | --- |
| `id` | `bigIncrements` | Chave primária |
| `nome` | `string` | Nome completo do contratante |
| `cpf_cnpj` | `string` | Documento (CPF ou CNPJ), único |
| `telefone` | `string` | Telefone para contato |
| `email` | `string` | E-mail, único |
| `numero_convidados` | `unsignedInteger` | Número estimado de convidados |
| `created_at` / `updated_at` | `timestamp` | Controlados pelo Eloquent |

### 4.2 CRUD via Controller

O CRUD completo é implementado em `App\Http\Controllers\ContratanteController`, com os métodos:

```php
index()    // GET    /contratantes
create()   // GET    /contratantes/create
store()    // POST   /contratantes
show()     // GET    /contratantes/{contratante}
edit()     // GET    /contratantes/{contratante}/edit
update()   // PUT    /contratantes/{contratante}
destroy()  // DELETE /contratantes/{contratante}
```

`store()` e `update()` recebem, respectivamente, `StoreContratanteRequest` e `UpdateContratanteRequest`.

### 4.3 FormRequest de validação

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreContratanteRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'nome' => ['required', 'string', 'max:255'],
            'cpf_cnpj' => ['required', 'string', 'max:18', 'unique:contratantes,cpf_cnpj'],
            'telefone' => ['required', 'string', 'max:20'],
            'email' => ['required', 'email', 'max:255', 'unique:contratantes,email'],
            'numero_convidados' => ['required', 'integer', 'min:1'],
        ];
    }
}
```

A validação de dígitos do CPF/CNPJ fica em uma regra customizada `App\Rules\CpfCnpjValido`, aplicada ao campo `cpf_cnpj`. No `UpdateContratanteRequest`, as regras `unique` ignoram o registro atual com `Rule::unique('contratantes', 'cpf_cnpj')->ignore($this->contratante)`.

## 5. Models Produto e Serviço

### 5.1 Estrutura das tabelas `produtos` e `servicos`

Campos obrigatórios em ambas:

| Coluna | Tipo PHP/DB | Descrição |
| --- | --- | --- |
| `id` | `bigIncrements` | Chave primária |
| `nome` | `string` | Nome do produto ou serviço |
| `descricao` | `text` | Descrição detalhada |
| `tipo` | `string` | `produto` ou `serviço` |
| `categoria_id` | `foreignId` | Chave estrangeira para `categorias` |
| `preco` | `decimal(10,2)` | Valor do item |
| `status` | `string` | `disponivel` ou `indisponivel` |

Campo complementar conforme o tipo:

- `App\Models\Servico` → `duracao_minutos` (`unsignedInteger`);
- `App\Models\Produto` → `estoque` (`unsignedInteger`).

> Regra de negócio: itens com `status = 'indisponivel'` não aparecem para seleção em um agendamento.

### 5.2 Itens do catálogo

Itens previstos para o catálogo da Villa Prime (classificação sugerida):

| Item | Model | Tipo | Campo complementar |
| --- | --- | --- | --- |
| Locação do salão | `Servico` | serviço | `duracao_minutos` |
| Buffet completo | `Servico` | serviço | `duracao_minutos` |
| Decoração temática | `Servico` | serviço | `duracao_minutos` |
| Som/iluminação | `Servico` | serviço | `duracao_minutos` |
| Doces e lembranças | `Produto` | produto | `estoque` |

### 5.3 FormRequests

`StoreProdutoRequest`, `UpdateProdutoRequest`, `StoreServicoRequest` e `UpdateServicoRequest` garantem a presença de `nome`, `descricao`, `tipo`, `categoria_id`, `preco` e `status`, além do campo condicional (`estoque` ou `duracao_minutos`).

### 5.4 Filtro e ordenação no catálogo

`CatalogoController::index()` monta a consulta com Query Builder/Eloquent:

```php
$itens = Produto::query()
    ->when($request->categoria, fn ($q) => $q->where('categoria_id', $request->categoria))
    ->orderBy('preco', $request->ordenar === 'preco_desc' ? 'desc' : 'asc')
    ->get();
```

O parâmetro `ordenar` é mapeado para valores fixos (`preco_asc` / `preco_desc`), evitando concatenar entrada do usuário na cláusula `ORDER BY`.

Exemplo: `GET /catalogo?categoria=alimentacao&ordenar=preco_asc`.

## 6. Model Agendamento

O **Agendamento** reserva uma data para um evento e associa o contratante aos produtos/serviços escolhidos.

### 6.1 Estrutura da tabela `agendamentos` (migration)

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('agendamentos', function (Blueprint $table) {
            $table->id();
            $table->foreignId('contratante_id')->constrained()->cascadeOnDelete();
            $table->date('data_evento')->unique();
            $table->time('horario');
            $table->string('status', 20)->default('pendente');
            $table->decimal('valor_total', 10, 2)->default(0);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('agendamentos');
    }
};
```

Tabelas pivô:

```php
// agendamento_produto
$table->foreignId('agendamento_id')->constrained()->cascadeOnDelete();
$table->foreignId('produto_id')->constrained()->cascadeOnDelete();
$table->unsignedInteger('quantidade');
$table->decimal('preco_unitario', 10, 2);

// agendamento_servico
$table->foreignId('agendamento_id')->constrained()->cascadeOnDelete();
$table->foreignId('servico_id')->constrained()->cascadeOnDelete();
$table->unsignedInteger('quantidade');
$table->decimal('preco_unitario', 10, 2);
```

### 6.2 Relacionamentos Eloquent

```php
<?php

namespace App\Models;

use App\Enums\StatusAgendamento;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Agendamento extends Model
{
    protected $fillable = [
        'contratante_id',
        'data_evento',
        'horario',
        'status',
        'valor_total',
    ];

    protected $casts = [
        'data_evento' => 'date',
        'valor_total' => 'decimal:2',
        'status' => StatusAgendamento::class,
    ];

    public function contratante(): BelongsTo
    {
        return $this->belongsTo(Contratante::class);
    }

    public function produtos(): BelongsToMany
    {
        return $this->belongsToMany(Produto::class, 'agendamento_produto')
            ->withPivot(['quantidade', 'preco_unitario']);
    }

    public function servicos(): BelongsToMany
    {
        return $this->belongsToMany(Servico::class, 'agendamento_servico')
            ->withPivot(['quantidade', 'preco_unitario']);
    }
}
```

### 6.3 Status em enum PHP

```php
<?php

namespace App\Enums;

enum StatusAgendamento: string
{
    case Pendente = 'pendente';
    case Confirmado = 'confirmado';
    case Concluido = 'concluido';
    case Cancelado = 'cancelado';
}
```

No banco, o valor gravado é `concluido` (sem acento). A interface exibe **Concluído**.

### 6.4 CRUD via Controller

`App\Http\Controllers\AgendamentoController` implementa `index`, `create`, `store`, `show`, `edit`, `update` e `destroy`. `store()` e `update()` usam `StoreAgendamentoRequest` / `UpdateAgendamentoRequest` e chamam `calcularValorTotal()` antes de persistir.

### 6.5 FormRequest do agendamento

```php
<?php

namespace App\Http\Requests;

use App\Enums\StatusAgendamento;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Enum;

class StoreAgendamentoRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        $id = $this->route('agendamento')?->id;

        return [
            'contratante_id' => ['required', 'exists:contratantes,id'],
            'data_evento' => ['required', 'date', Rule::unique('agendamentos', 'data_evento')->ignore($id)],
            'horario' => ['required', 'date_format:H:i'],
            'status' => ['required', new Enum(StatusAgendamento::class)],
            'produtos' => ['array'],
            'produtos.*.quantidade' => ['integer', 'min:1'],
            'servicos' => ['array'],
            'servicos.*.quantidade' => ['integer', 'min:1'],
        ];
    }
}
```

A regra `unique` garante que **não exista mais de um agendamento no mesmo dia**; ao editar, o próprio registro é ignorado.

### 6.6 Cálculo do valor total

```php
public function calcularValorTotal(): float
{
    $total = 0;

    foreach ($this->produtos as $produto) {
        $total += $produto->pivot->preco_unitario * $produto->pivot->quantidade;
    }

    foreach ($this->servicos as $servico) {
        $total += $servico->pivot->preco_unitario * $servico->pivot->quantidade;
    }

    return $this->valor_total = round($total, 2);
}
```

No formulário, o valor total é atualizado via JavaScript conforme os itens são selecionados; no servidor, o valor é recalculado e salvo, sem confiar no valor enviado pelo navegador.

## 7. Regras de negócio

1. **CPF/CNPJ válido:** validado por `App\Rules\CpfCnpjValido` e único em `contratantes.cpf_cnpj`.
2. **E-mail válido:** `email` + `unique:contratantes,email`.
3. **Telefone válido:** formato com DDD e número validado no FormRequest.
4. **Número estimado de convidados:** `integer` e `min:1`.
5. **Um agendamento por dia:** `unique` em `agendamentos.data_evento`.
6. **Itens disponíveis:** apenas `status = 'disponivel'` podem ser alocados.
7. **Valor total:** calculado no model e confirmado no servidor.
8. **Status controlado:** somente os casos de `StatusAgendamento`.

## 8. Segurança em PHP/Laravel

### 8.1 CSRF

Middleware `Illuminate\Foundation\Http\Middleware\VerifyCsrfToken` ativo nas rotas `web`. Todo formulário Blade inclui:

```blade
<form method="POST" action="{{ route('agendamentos.store') }}">
    @csrf
    ...
</form>
```

Em requisições AJAX, o token é enviado no cabeçalho `X-CSRF-TOKEN`.

### 8.2 XSS

Escape automático do Blade com `{{ $valor }}` (equivalente a `e($valor)` / `htmlspecialchars`). Não usar `{!! $valor !!}` com dados vindos do usuário. Textos são validados nos FormRequests antes da persistência.

### 8.3 SQL Injection

Consultas com Eloquent/Query Builder usam *prepared statements* do PDO. Não há concatenação de inputs em SQL. O filtro e a ordenação são mapeados para colunas e direções fixas:

```php
$direcao = $request->ordenar === 'preco_desc' ? 'desc' : 'asc';
$itens = Produto::orderBy('preco', $direcao)->get();
```

## 9. Estrutura de diretórios PHP

```text
app/
├── Enums/
│   └── StatusAgendamento.php
├── Http/
│   ├── Controllers/
│   │   ├── ContratanteController.php
│   │   ├── ProdutoController.php
│   │   ├── ServicoController.php
│   │   ├── CatalogoController.php
│   │   └── AgendamentoController.php
│   └── Requests/
│       ├── StoreContratanteRequest.php
│       ├── UpdateContratanteRequest.php
│       ├── StoreProdutoRequest.php
│       ├── UpdateProdutoRequest.php
│       ├── StoreServicoRequest.php
│       ├── UpdateServicoRequest.php
│       ├── StoreAgendamentoRequest.php
│       └── UpdateAgendamentoRequest.php
├── Models/
│   ├── Contratante.php
│   ├── Produto.php
│   ├── Servico.php
│   ├── Categoria.php
│   └── Agendamento.php
├── Rules/
│   └── CpfCnpjValido.php
└── Providers/
database/
├── factories/
├── migrations/
│   ├── ..._create_contratantes_table.php
│   ├── ..._create_categorias_table.php
│   ├── ..._create_produtos_table.php
│   ├── ..._create_servicos_table.php
│   ├── ..._create_agendamentos_table.php
│   ├── ..._create_agendamento_produto_table.php
│   └── ..._create_agendamento_servico_table.php
└── seeders/
resources/
└── views/
    ├── contratantes/
    ├── produtos/
    ├── servicos/
    ├── agendamentos/
    └── catalogo/
routes/
└── web.php
composer.json
.env.example
artisan
```

## 10. Rotas em `routes/web.php`

```php
use App\Http\Controllers\AgendamentoController;
use App\Http\Controllers\CatalogoController;
use App\Http\Controllers\ContratanteController;
use App\Http\Controllers\ProdutoController;
use App\Http\Controllers\ServicoController;
use Illuminate\Support\Facades\Route;

Route::resource('contratantes', ContratanteController::class);
Route::resource('agendamentos', AgendamentoController::class);
Route::resource('produtos', ProdutoController::class)->except(['show']);
Route::resource('servicos', ServicoController::class)->except(['show']);
Route::get('catalogo', [CatalogoController::class, 'index'])->name('catalogo.index');
```

| Método | Rota | Método do Controller |
| --- | --- | --- |
| GET | `/contratantes` | `index` |
| GET | `/contratantes/create` | `create` |
| POST | `/contratantes` | `store` |
| GET | `/contratantes/{contratante}` | `show` |
| GET | `/contratantes/{contratante}/edit` | `edit` |
| PUT/PATCH | `/contratantes/{contratante}` | `update` |
| DELETE | `/contratantes/{contratante}` | `destroy` |
| GET | `/agendamentos` | `index` |
| POST | `/agendamentos` | `store` |
| PUT/PATCH | `/agendamentos/{agendamento}` | `update` |
| DELETE | `/agendamentos/{agendamento}` | `destroy` |
| GET | `/catalogo` | `CatalogoController::index` |

## 11. Como executar

Pré-requisitos: **PHP 8.1+**, **Composer** e banco de dados (SQLite, MySQL ou PostgreSQL).

```bash
composer install
cp .env.example .env
php artisan key:generate
# Ajustar DB_* no .env
php artisan migrate
# Opcional: popular o catálogo
php artisan db:seed
php artisan serve
```

A aplicação fica disponível em `http://127.0.0.1:8000`.

## 12. Checklist de implementação

- [ ] `composer.json` e configuração do Laravel
- [ ] Migrations: `contratantes`, `categorias`, `produtos`, `servicos`, `agendamentos` e tabelas pivô
- [ ] Models com `$fillable`, `$casts` e relacionamentos Eloquent
- [ ] `StatusAgendamento` em `app/Enums` e `CpfCnpjValido` em `app/Rules`
- [ ] FormRequests de contratantes, produtos, serviços e agendamentos
- [ ] Resource controllers com os sete métodos
- [ ] CRUD completo de contratantes
- [ ] Catálogo com filtro por categoria e ordenação por preço
- [ ] CRUD completo de agendamentos com data, horário e status
- [ ] `unique` em `data_evento` e validação no FormRequest
- [ ] `calcularValorTotal()` e exibição do total na seleção de itens
- [ ] `@csrf`, escaping Blade e consultas com bindings
- [ ] Views Blade para todas as operações

## 13. Roadmap

1. Criar migrations e models de contratantes, catálogo e agendamentos.
2. Implementar FormRequests e regras customizadas de CPF/CNPJ.
3. Implementar os resource controllers e as rotas em `web.php`.
4. Implementar catálogo com filtro e ordenação.
5. Implementar agendamento com unicidade de data e cálculo do valor total.
6. Aplicar proteções CSRF, XSS e SQL injection.
7. Refinar as views Blade e a experiência do usuário.
