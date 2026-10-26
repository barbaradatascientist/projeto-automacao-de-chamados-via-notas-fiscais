# Guia de Contribuição Interna 🚀

Este documento serve como um guia de referência rápida para garantir a consistência do código, a execução dos testes e o fluxo correto de branches no repositório, já que o projeto é mantido por uma única pessoa.

---

## 🛠️ Fluxo de Trabalho Diário (Passo a Passo)

Sempre que for iniciar uma nova tarefa, correção ou funcionalidade, siga rigorosamente os passos abaixo:

### 1. Atualizar a Branch de Desenvolvimento
Antes de criar qualquer branch, garanta que sua base local está idêntica ao GitHub.
```bash
git checkout develop
git pull origin develop
```

### 2. Criar uma Branch de Recurso (Feature)
Nunca trabalhe direto na `develop` ou na `main`. Crie uma branch específica para o que vai fazer:
```bash
# Para novas funcionalidades: feature/nome-da-tarefa
# Para correções de bugs: bugfix/nome-do-erro
git checkout -b feature/minha-nova-tela
```

### 3. Iniciar o Ambiente Virtual
Sempre ative o ambiente virtual para garantir que as dependências e ferramentas estão isoladas.
```bash
# No Windows (PowerShell):
.venv\Scripts\Activate.ps1

# No Windows (Prompt de comando comum):
.venv\Scripts\activate

# No Mac/Linux:
source .venv/bin/activate
```

### 4. Desenvolver e Validar Localmente
Escreva o seu código. Antes de enviar para o GitHub, rode as ferramentas de qualidade para evitar que o robô do GitHub Actions rejeite o seu código:

```bash
# 1. Corrigir erros de estilo/linting automaticamente
ruff check --fix

# 2. Formatar o código no padrão
ruff format

# 3. Rodar a suite de testes locais
pytest
```

### 5. Atualizar as Dependências (Se necessário)
Se você instalou alguma biblioteca nova via `pip` durante o desenvolvimento, atualize o arquivo de requisitos:
```bash
pip freeze > requirements.txt
```

### 6. Enviar as Alterações (Push)
Adicione os arquivos alterados, faça o commit e envie a branch para o repositório remoto:
```bash
git add .
git commit -m "feat: adiciona nova funcionalidade XPTO"
git push origin feature/minha-nova-tela
```

---

## 🔀 Fluxo de Integração (Pull Requests)

### Da Feature para a `develop`
1. Vá até o GitHub e abra um Pull Request (PR) da sua branch (`feature/*`) direcionado para a **`develop`**.
2. O GitHub Actions vai rodar automaticamente o **Ruff** e o **Pytest**.
3. **Se os testes passarem (Verde):** Você está liberada para fazer o Merge do PR para a `develop`.
4. **Se os testes falharem (Vermelho):** Corrija os erros na sua máquina, faça um novo `git push` e espere o robô validar novamente.

### Da `develop` para a `main` (Produção)
Quando terminar um ciclo de desenvolvimento e quiser lançar uma versão estável:
1. Abra um PR da **`develop`** direcionado para a **`main`**.
2. Aguarde a validação dos testes pelo GitHub Actions.
3. Faça o merge para publicar as alterações em produção.

---

## 🛑 Lembretes Importantes
* **Nunca** desative o ambiente virtual (`.venv`) enquanto estiver codificando.
* **Nunca** remova o `.venv/` do arquivo `.gitignore`. Essa pasta jamais deve subir para o GitHub.
* Mensagens de commit limpas ajudam na organização (Ex: `feat: ...`, `fix: ...`, `refactor: ...`, `ci: ...`).
