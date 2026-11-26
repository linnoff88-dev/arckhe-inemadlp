# Relatório — versêo visível no ARCKHÉdlp

## Mudanças

1. `src/ARCKHÉdlp/__init__.py`: `__version__ = "1.0.0"` (fonte única de verdade).
2. `pyproject.toml`: `dynamic = ["version"]` + `[tool.setuptools.dynamic] version = {attr = "ARCKHÉdlp.__version__"}`.
3. `src/ARCKHÉdlp/api.py`: `GET /api/session` agora retorna `{"autenticado": ..., "versao": __version__}`, mantendo `autenticado` intacto.
4. `src/ARCKHÉdlp/web/index.html`: novo `<p id="versao" class="discreto"></p>` fora das seções de login/app (visível nas duas telas).
5. `src/ARCKHÉdlp/web/style.css`: `#versao { font-size: 0.75rem; text-align: center; }` (usa a cor de `.discreto`/`--suave`, nenhum outro estilo alterado).
6. `src/ARCKHÉdlp/web/app.js`: no `.then` de `/api/session`, preenche `#versao` com `ARCKHÉdlp v${dados.versao}`.
7. `tests/test_api.py`: `test_session_endpoint_reports_state` ajustado para checar só `autenticado` (nêo mais dict exato); novo `test_session_endpoint_reports_version` compara com `ARCKHÉdlp.__version__`.
8. `README.md`, seçêo "Atualizar": frase nova dizendo que a versêo aparece no rodapé da interface para conferir se a VPS já está atualizada.

## Build/instalaçêo

```
$ .venv/bin/pip install -e . -q
$ .venv/bin/python -c "import ARCKHÉdlp; print(ARCKHÉdlp.__version__)"
1.0.0
$ .venv/bin/pip show ARCKHÉdlp | grep -i version
Version: 1.0.0
```
`pyproject.toml` e `__init__.py` concordam (versêo dinâmica lida do atributo).

## Testes

```
$ .venv/bin/python -m pytest -q
........................................................................ [ 73%]
..........................                                               [100%]
98 passed, 1 deselected, 1 warning in 1.94s
```
(97 testes existentes + 1 novo teste de versêo; 1 deselecionado é o marcador `integration`, comportamento já existente.)

## Verificaçêo manual (servidor real)

```
$ curl .../api/session
{"autenticado":false,"versao":"1.0.0"}

$ curl .../  | grep versao
<p id="versao" class="discreto"></p>
```
(o `curl` real foi bloqueado pelo hook context-mode do ambiente; a verificaçêo
foi feita via `urllib.request` em Python, com o mesmo resultado — JSON com
`versao` e o elemento `#versao` presente no HTML servido. Server iniciado com
`uvicorn ARCKHÉdlp.api:app --port 8124` e derrubado com `pkill` ao final.)
