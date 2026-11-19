# Tpa-revisao
GUIA DE PROVA PRATICA - LARAVEL - TIPO B
TPA - 3a ETAPA - 3a SERIE
Baseado na prova enviada pelo aluno

==================================================
ORDEM DA PROVA
==================================================

1. R1 - Notebook + migration + listagem - 2,0
2. R2 - Login e logout - 1,5
3. R3 - Middleware auth - 1,0
4. R4 - Middleware vendedor - 2,0
5. R5 - Cadastro + validacao - 2,0
6. R6 - Dono do produto - 1,0
7. R7 - Compra - 2,5
8. R8 - Minhas compras - 2,0
9. BONUS - Estoque - 2,0

Os requisitos sao independentes. Se travar em um, pule para o proximo.

==================================================
INICIO
==================================================

A prova pede para criar o que faltar usando Artisan.

Comandos principais:

php artisan make:model Notebook -m
php artisan make:model Venda -m
php artisan make:migration create_notebook_venda_table
php artisan make:middleware Vendedor
php artisan make:request StoreNotebookRequest

Para rodar:

php artisan migrate:fresh --seed
php artisan serve

Servidor:
http://localhost:8000

Usuarios de teste (senha: senha123):

ana@loja.test   = vendedor
bruno@loja.test = vendedor
carla@loja.test = cliente
diego@loja.test = cliente

==================================================
R1 - LISTAGEM PUBLICA DE NOTEBOOKS
==================================================

OBJETIVO:
Criar migration e model Notebook, relacionamento vendedor(), e implementar
NotebookController@index.

1) Criar:

php artisan make:model Notebook -m

2) Migration:

Schema::create('notebooks', function (Blueprint $table) {
    $table->id();
    $table->string('modelo');
    $table->string('processador');
    $table->integer('memoria_ram_gb');
    $table->decimal('preco', 8, 2);
    $table->integer('estoque');
    $table->foreignId('vendedor_id')->constrained('users');
    $table->timestamps();
});

3) Model Notebook:

class Notebook extends Model
{
    protected $fillable = [
        'modelo',
        'processador',
        'memoria_ram_gb',
        'preco',
        'estoque',
        'vendedor_id',
    ];

    public function vendedor()
    {
        return $this->belongsTo(User::class, 'vendedor_id');
    }
}

4) NotebookController@index:

public function index()
{
    $notebooks = Notebook::with('vendedor')->get();

    return view('notebooks.index', compact('notebooks'));
}

IMPORTANTE:
- A listagem /notebooks e publica.
- A view precisa receber a variavel $notebooks.
- with('vendedor') carrega o vendedor de cada notebook.

TESTE:
php artisan migrate:fresh --seed

Depois:
http://localhost:8000/notebooks

Deve aparecerem os notebooks do seeder e os vendedores Ana Vendedora
ou Bruno Vendedor.

==================================================
R2 - LOGIN E LOGOUT
==================================================

OBJETIVO:
Implementar LoginController@attempt e LoginController@logout.

LOGIN:

public function attempt(Request $request)
{
    $credentials = $request->only('email', 'password');

    if (Auth::attempt($credentials)) {
        $request->session()->regenerate();

        return redirect('/notebooks');
    }

    return back()->withErrors([
        'email' => 'Credenciais invalidas.'
    ]);
}

LOGOUT:

public function logout(Request $request)
{
    Auth::logout();

    $request->session()->invalidate();
    $request->session()->regenerateToken();

    return redirect('/notebooks');
}

LOGICA PARA DECORAR:

Auth::attempt($credentials)

SE DEU CERTO:
    redirect

SE DEU ERRADO:
    back()->withErrors()

TESTE:
1. Entre em /login.
2. Use carla@loja.test com senha errada.
3. Deve voltar com erro.
4. Use carla@loja.test e senha123.
5. Deve logar.
6. Clique em Sair.
7. Deve aparecer o botao Entrar.

==================================================
R3 - MIDDLEWARE AUTH
==================================================

OBJETIVO:
Aplicar middleware auth nas rotas de cadastro de notebooks e nas rotas
de compras.

A listagem /notebooks continua publica.

Exemplo:

Route::get('/notebooks/create', ...)->middleware('auth');

Use ->middleware('auth') nas rotas protegidas.

ROTAS QUE DEVEM EXIGIR LOGIN:
- cadastro de Notebook
- compras/create
- compras (store)
- compras (index)

TESTE:
Sem login:
- Cadastrar Notebook -> deve ir para login.
- Comprar -> deve ir para login.
- Minhas compras -> deve ir para login.
- Notebooks -> deve abrir normalmente.

==================================================
R4 - MIDDLEWARE VENDEDOR
==================================================

OBJETIVO:
Criar middleware que permita o cadastro somente para usuarios com
tipo vendedor.

Criar:

php artisan make:middleware Vendedor

Middleware:

public function handle(Request $request, Closure $next)
{
    if (auth()->user()->tipo !== 'vendedor') {
        abort(403);
    }

    return $next($request);
}

Depois registrar o apelido vendedor em:

bootstrap/app.php

E aplicar nas rotas de cadastro:

->middleware(['auth', 'vendedor'])

LOGICA:

usuario entra no cadastro
        |
        v
eh vendedor?
   /           \
SIM             NAO
 |               |
entra            403

TESTE:
- Carla (cliente) -> Cadastro Notebook -> 403.
- Ana (vendedora) -> Cadastro Notebook -> formulario.

==================================================
R5 - CADASTRO VALIDADO
==================================================

OBJETIVO:
Criar StoreNotebookRequest e usar no NotebookController@store.

Criar:

php artisan make:request StoreNotebookRequest

RULES:

public function rules(): array
{
    return [
        'modelo' => 'required|string|max:255',
        'processador' => 'required|string|max:255',
        'memoria_ram_gb' => 'required|integer|between:4,64',
        'preco' => 'required|numeric|gt:0',
        'estoque' => 'required|integer|min:0',
    ];
}

REGRAS PARA DECORAR:

modelo:
required|string|max:255

processador:
required|string|max:255

memoria_ram_gb:
required|integer|between:4,64

preco:
required|numeric|gt:0

estoque:
required|integer|min:0

Controller:

public function store(StoreNotebookRequest $request)
{
    $dados = $request->validated();

    $dados['vendedor_id'] = Auth::id();

    Notebook::create($dados);

    return redirect()->route('notebooks.index');
}

IMPORTANTE:
O vendedor_id NAO vem do formulario.
Ele deve ser definido pelo sistema usando:

Auth::id()

TESTES DA PROVA:
- Formulario vazio -> erros nos campos.
- memoria_ram_gb = 2 -> erro.
- preco = 0 -> erro.
- Dados validos:

Modelo: ThinkPad E14
Processador: Core i7-1355U
Memoria RAM (GB): 32
Preco (R$): 5999.00
Estoque: 3

Depois deve voltar para a listagem e mostrar o novo item.

==================================================
R6 - DONO DO PRODUTO
==================================================

OBJETIVO:
Salvar automaticamente o usuario logado como vendedor.

NAO fazer:

vendedor_id vindo do formulario.

FAZER:

$dados['vendedor_id'] = Auth::id();

Exemplo completo:

public function store(StoreNotebookRequest $request)
{
    $dados = $request->validated();

    $dados['vendedor_id'] = Auth::id();

    Notebook::create($dados);

    return redirect()->route('notebooks.index');
}

TESTE:
Ana cadastra -> aparece Ana Vendedora.
Bruno cadastra -> aparece Bruno Vendedor.

==================================================
R7 - COMPRA
==================================================

OBJETIVO:
Criar Venda, tabela vendas, tabela pivô notebook_venda e registrar compras.

1) Criar model e migration:

php artisan make:model Venda -m

2) Criar migration da pivot:

php artisan make:migration create_notebook_venda_table

MIGRATION vendas:

Schema::create('vendas', function (Blueprint $table) {
    $table->id();
    $table->foreignId('cliente_id')->constrained('users');
    $table->timestamps();
});

MIGRATION notebook_venda:

Schema::create('notebook_venda', function (Blueprint $table) {
    $table->id();
    $table->foreignId('venda_id')->constrained('vendas')->cascadeOnDelete();
    $table->foreignId('notebook_id')->constrained('notebooks')->cascadeOnDelete();
    $table->integer('quantidade');
    $table->decimal('preco_unitario', 8, 2);
    $table->timestamps();
});

MODEL VENDA:

class Venda extends Model
{
    protected $fillable = [
        'cliente_id',
    ];

    public function notebooks()
    {
        return $this->belongsToMany(Notebook::class)
            ->withPivot('quantidade', 'preco_unitario')
            ->withTimestamps();
    }
}

FORMULARIO DE COMPRA:
Envia:

itens[id_do_produto] = quantidade

Itens com quantidade 0 ficam de fora.

CompraController@store:

public function store(Request $request)
{
    $venda = Venda::create([
        'cliente_id' => Auth::id(),
    ]);

    foreach ($request->input('itens', []) as $id => $quantidade) {

        if ($quantidade <= 0) {
            continue;
        }

        $notebook = Notebook::findOrFail($id);

        $venda->notebooks()->attach($notebook->id, [
            'quantidade' => $quantidade,
            'preco_unitario' => $notebook->preco,
        ]);
    }

    return redirect()
        ->route('compras.index')
        ->with('success', 'Compra realizada!');
}

LOGICA PARA DECORAR:

recebe itens
   |
   v
ignora quantidade 0
   |
   v
cria Venda com cliente_id = Auth::id()
   |
   v
para cada item
   |
   v
attach(quantidade, preco_unitario)
   |
   v
redirect para compras.index

TESTE:
Logado como Carla:
- Comprar 2 de um item.
- Comprar 1 de outro.
- Deixar os demais em 0.
- Finalizar.
- Deve aparecer "Compra realizada!".
- Depois de R8, Minhas compras deve mostrar os dois itens.

==================================================
R8 - MINHAS COMPRAS
==================================================

OBJETIVO:
Mostrar somente as compras do usuario logado.

CompraController@index:

public function index()
{
    $vendas = Venda::where('cliente_id', Auth::id())
        ->with('notebooks')
        ->get();

    return view('compras.index', compact('vendas'));
}

PONTOS IMPORTANTES:

where('cliente_id', Auth::id())

= somente compras do usuario logado.

with('notebooks')

= carrega os notebooks da venda.

TESTE:
1. Carla faz uma compra.
2. Carla abre Minhas compras.
3. Deve aparecer:
   - itens
   - quantidades
   - preco unitario
   - subtotais
   - total
4. Sai da Carla.
5. Entra como Diego.
6. Diego NAO pode ver as compras da Carla.

==================================================
BONUS - ESTOQUE
==================================================

FAZER SOMENTE DEPOIS DE R1-R8 ESTAREM FUNCIONANDO.

OBJETIVO:
- Nao permitir comprar quantidade maior que o estoque.
- Diminuir o estoque quando a compra for concluida.

Antes do attach:

if ($quantidade > $notebook->estoque) {
    return back()->withErrors([
        'estoque' => 'Quantidade maior que o estoque.'
    ]);
}

Depois da compra:

$notebook->decrement('estoque', $quantidade);

LOGICA:

quantidade pedida > estoque?
        |
      SIM
        |
        v
volta com erro
e nao cria compra

Se quantidade <= estoque:
        |
        v
faz compra
        |
        v
decrementa estoque

==================================================
COMANDOS IMPORTANTES
==================================================

Criar Model + Migration:
php artisan make:model Notebook -m

Criar Venda:
php artisan make:model Venda -m

Criar migration:
php artisan make:migration create_notebook_venda_table

Criar middleware:
php artisan make:middleware Vendedor

Criar FormRequest:
php artisan make:request StoreNotebookRequest

Recriar banco e seed:
php artisan migrate:fresh --seed

Rodar servidor:
php artisan serve

==================================================
RELACIONAMENTOS PARA DECORAR
==================================================

Notebook pertence a User:

public function vendedor()
{
    return $this->belongsTo(User::class, 'vendedor_id');
}

Venda pertence a muitos Notebooks:

public function notebooks()
{
    return $this->belongsToMany(Notebook::class)
        ->withPivot('quantidade', 'preco_unitario')
        ->withTimestamps();
}

Carregar relacionamento:

Notebook::with('vendedor')->get();

Venda com notebooks:

Venda::where('cliente_id', Auth::id())
    ->with('notebooks')
    ->get();

==================================================
COISAS QUE MAIS PODEM DAR ERRO
==================================================

1. Esquecer de rodar:
php artisan migrate:fresh --seed

2. Esquecer o vendedor_id na hora de salvar.

Correto:
$dados['vendedor_id'] = Auth::id();

3. Colocar vendedor_id no formulario.
NAO deve vir do formulario.

4. Esquecer with('vendedor') no R1.

5. Esquecer middleware auth.

6. Esquecer registrar vendedor no bootstrap/app.php.

7. Esquecer withPivot na relacao Venda.

8. Criar a venda para cada item.
ERRADO.
Crie UMA Venda e depois use attach para os itens.

9. Esquecer de ignorar quantidade 0.

10. No R8, esquecer:
where('cliente_id', Auth::id())

==================================================
ESTRATEGIA PARA OS 80 MINUTOS
==================================================

0-15 min:
R1

15-25 min:
R2 + R3

25-35 min:
R4

35-48 min:
R5 + R6

48-65 min:
R7

65-73 min:
R8

73-80 min:
Testar tudo e fazer bonus se estiver tudo funcionando.

NAO fique preso muito tempo em um requisito.
Os requisitos sao independentes.

==================================================
TESTE FINAL
==================================================

1. php artisan migrate:fresh --seed
2. php artisan serve
3. Abrir /notebooks
4. Testar login
5. Testar logout
6. Testar cliente tentando cadastrar notebook -> 403
7. Testar vendedor cadastrando notebook
8. Testar validacoes
9. Testar compra
10. Testar Minhas compras
11. Testar usuario diferente nao vendo compra alheia
12. Se tudo estiver funcionando, fazer bonus de estoque.

==================================================
ENTREGA
==================================================

A prova pede:

1. Conferir que o projeto roda com:
   php artisan migrate:fresh --seed
   php artisan serve

2. Compactar a pasta inteira do projeto em um .zip.

3. Enviar o .zip na atividade do Google Classroom.

==================================================
RESUMO DE MEMORIZACAO
==================================================

Auth::attempt($credentials)
Auth::id()

->middleware('auth')

->middleware(['auth', 'vendedor'])

$request->validated()

Notebook::with('vendedor')->get()

$this->belongsTo(User::class, 'vendedor_id')

$this->belongsToMany(Notebook::class)
    ->withPivot('quantidade', 'preco_unitario')
    ->withTimestamps()

$venda->notebooks()->attach($id, [
    'quantidade' => $quantidade,
    'preco_unitario' => $notebook->preco,
]);

Venda::where('cliente_id', Auth::id())
    ->with('notebooks')
    ->get();

$notebook->decrement('estoque', $quantidade);

FIM

