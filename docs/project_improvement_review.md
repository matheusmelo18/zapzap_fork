# Relatório de Análise e Sugestões de Melhorias no Projeto ZapZap

## 1. Resumo Executivo

O **ZapZap** é um aplicativo desktop maduro, bem estruturado e focado no usuário, desenvolvido em Python 3 e PyQt6 com suporte nativo a múltiplos perfis do QtWebEngine. A arquitetura orientada a camadas (camada principal `app`, domínios semânticos em `core`, primitivas e componentes reutilizáveis em `ui` e módulos isolados em `features`) demonstra excelente disciplina de design e clareza de responsabilidades.

Após uma auditoria completa do repositório — cobrindo código-fonte, testes, automação de CI, documentação técnica e internacionalização —, identificamos **oportunidades chave de melhoria** para elevar a qualidade do código, estabilidade nos testes, eficiência de recursos e manutenibilidade contínua.

---

## 2. Arquitetura e Organização do Código

### 2.1. Eliminação de Código Morto e Refatoração de Modelos
- **Identificação de Código Não Utilizado**: O script interno de análise estática (`tests/check_unused_code.py`) aponta **65 ocorrências** de código sem referências ativas no projeto.
  - **Classes obsoletas**: `ExtensionRuntimeManager` (em `zapzap/features/customizations/extension_runtime_manager.py`) e `SettingsLabelRow` (em `zapzap/ui/components/settings_rows.py`).
  - **Importações sem uso**: `SystemIcon` em `downloads_menu.py`, `Qt` em `reporting/dialogs.py`, `QPoint` e `QSize` em `edit_account_dialog.py`.
  - **Métodos legados em modelos**: O modelo `SystemStartupSettingsModel` em `zapzap/features/settings/pages/system_startup/model.py` contém cópias obsoletas de métodos de gerenciamento de dicionários (`get_dictionaries_path`, `list_dictionaries`, `set_dictionary_language`, etc.) herdados de refatorações anteriores.
- **Recomendação**: Remover de forma sistemática todas as classes, importações e métodos mortos para manter a base de código enxuta e facilitar a manutenção.

### 2.2. Tratamento de Exceções Genéricas
- **Uso de `except:` sem especificação de tipo**: Localizado em `zapzap/features/browser/shell/browser_controller.py:875`. Capturar todas as exceções (incluindo `KeyboardInterrupt` e `SystemExit`) é uma má prática em Python.
- **Supressão silenciosa em blocos `except Exception:`**: Em áreas críticas como `web_view.py`, `settings_controller.py` e `performance.py`, alguns erros de runtime são ignorados silenciosamente sem log estruturado de diagnóstico.
- **Recomendação**: Substituir `except:` por exceções específicas (ex: `AttributeError`, `KeyError`, `PyQt6` errors) e assegurar que erros inesperados sejam devidamente registrados via `core.diagnostics` ou logger da aplicação.

---

## 3. Qualidade de Código, Tipagem e Padronização

### 3.1. Expansão das Anotações de Tipo (Type Hints)
- **Cenário atual**: Várias partes do projeto possuem boa tipagem, mas páginas de configurações em `features/settings/pages/` e componentes customizados de UI ainda contêm funções e retornos sem anotações.
- **Recomendação**:
  1. Padronizar as anotações de tipo usando `typing` / Python 3.10+ (ex: `list[str]`, `str | None`).
  2. Adicionar verificação estática de tipos (ex: `mypy` ou `pyright`) no pipeline de Integração Contínua (CI).

### 3.2. Adição de Linters Modernos (Ruff)
- **Cenário atual**: A verificação de estilo do projeto depende do `git diff --check` e do script `check_unused_code.py`.
- **Recomendação**: Adicionar o `ruff` ao projeto para verificação ultra-rápida de estilo, importações não utilizadas e regras PEP8.

---

## 4. Interface de Usuário (UI/UX) e Internacionalização (i18n)

### 4.1. Correções de Catálogos de Tradução
- **Falha no teste de tradução em turco**: O teste `test_turkish_translation.py` reporta que a string `"Strict proxy isolation will not be changed."` está não traduzida/fuzzy no catálogo `po/tr/LC_MESSAGES/zapzap.po`.
- **Validação de todos os catálogos no CI**: O workflow do GitHub Actions (`quality.yml`) valida explicitamente apenas o catálogo de turco.
- **Recomendação**:
  1. Atualizar a tradução em turco para corrigir a falha no teste.
  2. Expandir a validação automatizada de PO no CI para testar todos os idiomas listados em `po/LINGUAS`.

### 4.2. Acessibilidade e Leitores de Tela
- **Atributos de Acessibilidade**: O script `resizable_chat_list.js` inclui propriedades ARIA (`aria-valuenow`, `aria-valuemin`, `aria-valuemax`). No entanto, componentes visuais customizados PyQt6 (como cartões de contas e botões da sidebar) podem ser aprimorados com `setAccessibleName` e `setAccessibleDescription`.
- **Recomendação**: Auditar e definir nomes acessíveis em todos os componentes em `zapzap/ui/components/` e `zapzap/ui/primitives/`.

---

## 5. Desempenho, Memória e Ciclo de Vida do WebEngine

### 5.1. Otimização do Consumo de Memória RAM com Múltiplas Contas
- **Isolamento do QtWebEngine**: Cada conta ativa do WhatsApp roda em seu próprio perfil `QWebEngineProfile` e processo de renderização Chromium. Com 3 ou mais contas, o consumo de RAM pode crescer significativamente.
- **Recomendação**:
  1. Implementar/refinar a suspensão inteligente de contas em segundo plano quando a opção de economia de memória estiver habilitada.
  2. Garantir a liberação e destruição imediata de instâncias de `QWebEnginePage` ao desativar ou remover uma conta (`WebView.enable_page`).

---

## 6. Testes Automatizados e CI/CD

### 6.1. Ampliação do Escopo da Suíte de Testes no CI
- **Execução Parcial de Testes**: O workflow `.github/workflows/quality.yml` executa apenas um subconjunto de testes (como `test_download_settings.py`, `test_taskbar_badge.py`, etc.). A suíte completa tem mais de 40 arquivos de teste.
- **Integração de `check_unused_code.py` no CI**: O passo no CI roda `check_unused_code.py --packages-only`, ignorando a análise estática completa de código não utilizado.
- **Recomendação**:
  1. Incluir a execução do `check_unused_code.py` sem argumentos no CI de qualidade.
  2. Adicionar uma etapa no CI para rodar toda a suíte de testes unitários que não dependam obrigatoriamente de um servidor gráfico display real.

---

## 7. Matriz de Prioridades de Ação

| Prioridade | Categoria | Descrição da Melhoria | Esforço | Impacto |
| :---: | :---: | :--- | :---: | :---: |
| **Alta** | **i18n / Testes** | Atualizar catálogo turco (`po/tr/LC_MESSAGES/zapzap.po`) e validar todos os arquivos `.po` no CI | Baixo | Alto |
| **Alta** | **Qualidade** | Corrigir as 65 ocorrências de código morto e importações não utilizadas apontadas por `check_unused_code.py` | Médio | Alto |
| **Alta** | **Qualidade** | Substituir `except:` genérico e tratar exceções específicas com log adequado | Baixo | Médio |
| **Média** | **CI/CD** | Expandir `quality.yml` para rodar `check_unused_code.py` completo e linter `ruff` | Baixo | Alto |
| **Média** | **Performance** | Otimizar reciclagem de memória para contas inativas no `BrowserController` | Médio | Alto |
| **Média** | **Acessibilidade** | Adicionar propriedades acessíveis (`setAccessibleName`) em componentes de `ui/` | Médio | Médio |
| **Baixa** | **Tipagem** | Adicionar type hints completos e integrar `mypy`/`pyright` no CI | Alto | Médio |

---

*Documento gerado como parte da auditoria de qualidade do projeto ZapZap.*
