# Funcionalidade: Anexar documentos (PDF, Word, Excel, CSV)

Adiciona um botão de anexo (📎) na barra do editor Quill. Ao clicar, você escolhe um documento, ele faz upload para o servidor e insere no artigo um "chip" clicável (com ícone do tipo, nome do arquivo e tamanho). Ao clicar no chip no artigo público, abre em nova aba — PDF e CSV são exibidos pelo navegador; Word e Excel baixam (comportamento padrão do navegador, não há como exibir esses inline em localhost).

## Configurações aplicadas

- **Tipos aceitos:** pdf, xlsx, xls, docx, doc, csv
- **Tamanho máximo:** 100 MB por arquivo
- **Abertura:** nova aba (target="_blank")

LEMBRETE: você já ajustou o php.ini para `upload_max_filesize=100M` e `post_max_size=110M`. Sem isso, uploads grandes falhariam. Mantenha esses valores.

## Arquivos incluídos

```
app/Http/Controllers/Admin/UploadAnexoController.php   (NOVO)
public/js/editor-artigo.js                             (SUBSTITUI - agora com imagem + anexo)
public/css/anexos.css                                  (NOVO)
```

## PASSO 1 — Copiar os arquivos

Copie os 3 arquivos para o projeto, substituindo o `editor-artigo.js` (a nova versão mantém tudo que já fazia para imagens e soma o anexo).

## PASSO 2 — Adicionar a rota de upload de anexo

No `routes/web.php`, no topo junto aos outros `use`:

```php
use App\Http\Controllers\Admin\UploadAnexoController;
```

E dentro do grupo `admin` (perto da rota de upload de imagem que você já adicionou):

```php
Route::post('upload-anexo', [UploadAnexoController::class, 'store'])->name('upload.anexo');
```

## PASSO 3 — Passar a URL do anexo para o editor (views create e edit)

Nas views `resources/views/admin/artigos/create.blade.php` e `edit.blade.php`, localize a linha do editor:

```blade
<div id="editor-quill" data-upload-url="{{ route('admin.upload.imagem') }}"></div>
```

Adicione o atributo `data-anexo-url`:

```blade
<div id="editor-quill"
     data-upload-url="{{ route('admin.upload.imagem') }}"
     data-anexo-url="{{ route('admin.upload.anexo') }}"></div>
```

Faça isso nos DOIS arquivos (create e edit).

## PASSO 4 — Incluir o CSS dos anexos

### No admin (create.blade.php e edit.blade.php)

No bloco `@push('styles')`, logo após o `editor-artigo.css`, adicione:

```blade
<link rel="stylesheet" href="{{ asset('css/anexos.css') }}">
```

### No público (show.blade.php)

No bloco `@push('styles')`, logo após o `editor-artigo.css`, adicione a mesma linha:

```blade
<link rel="stylesheet" href="{{ asset('css/anexos.css') }}">
```

## PASSO 5 — Liberar os atributos do chip no Purifier

O chip de anexo usa `<a>` com `class`, `target`, `rel` e `data-ext`. O perfil `artigo` do Purifier já permite `<a href>`, mas precisamos garantir os atributos extras. Abra `config/purifier.php` e no perfil `artigo`, na linha `HTML.Allowed`, ajuste a parte do link `a[...]` para:

```php
'a[href|title|target|rel|class|data-ext]',
```

E, ainda no perfil `artigo`, adicione esta linha (permite o atributo data-ext):

```php
'HTML.AllowedAttributes' => '*.class,*.style,a.href,a.target,a.rel,a.data-ext,img.src,img.alt,img.width,img.height',
```

Se preferir não mexer muito, a forma mais simples e segura é adicionar ao perfil `artigo` a diretiva que habilita atributos `data-*`:

```php
'Attr.EnableID' => false,
'HTML.SafeIframe' => false,
%HTML.Allowed já contém a[...|class|data-ext] conforme acima
```

Depois de editar o purifier, rode:

```cmd
php artisan config:clear
```

OBS: se o `data-ext` for removido pelo Purifier mesmo assim, não tem problema funcional — ele só serve para a corzinha da borda lateral por tipo. O link continua funcionando. Se quiser garantir a cor sem depender do data-ext, me avise que troco a abordagem para classes (ex: anexo-pdf, anexo-word).

## PASSO 6 — storage:link (já está OK)

As imagens já funcionam, então o `storage:link` está correto. Os anexos vão para `storage/app/public/artigos/anexos`, dentro do mesmo link que já existe. Não precisa rodar de novo.

## PASSO 7 — Limpar caches

```cmd
cd C:\laragon\www\base-conhecimento
php artisan route:clear
php artisan view:clear
php artisan config:clear
```

Recarregue o admin com Ctrl+F5.

## Como usar

1. No editor de artigo, clique no botão de clipe (📎) na barra de ferramentas
2. Escolha um PDF, planilha ou documento Word
3. Aguarde o upload — aparece o chip com o nome e tamanho do arquivo
4. Salve o artigo
5. No artigo público, clique no chip: PDF/CSV abrem em nova aba, Word/Excel baixam

## Notas técnicas

- **Segurança:** a validação usa whitelist estrita de extensões/MIME (`mimes:pdf,xlsx,xls,docx,doc,csv`). Arquivos executáveis ou scripts são rejeitados. O nome físico no disco é gerado pelo sistema (evita path traversal); o nome original é preservado só no texto do link.
- **Nome do arquivo:** o usuário vê o nome original (ex: "Manual_SGI.pdf"), mas no disco ele é salvo como "2026-06-08_aB3xK9....pdf" para evitar colisões.
- **Por que Word/Excel não abrem inline:** o navegador não tem visualizador nativo desses formatos. Visualização inline exigiria um serviço externo (Google Docs Viewer/Office Online) que precisa do arquivo numa URL pública na internet — inviável em 127.0.0.1. Em produção com domínio público, seria possível adicionar isso futuramente.
- **Limpeza de órfãos:** anexos (como imagens) não são apagados do disco ao serem removidos do texto. Fica registrado para a futura rotina de limpeza de arquivos órfãos.
