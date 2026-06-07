# kiro-loja-infra

Repositório de infraestrutura compartilhada do projeto **kiro-loja**.

Contém a camada de fundação da plataforma: rede, balanceador de carga, cluster de
contêineres e repositórios de imagens. Nenhum serviço de aplicação é definido aqui —
cada serviço tem seu próprio repositório e stack CloudFormation que importa os recursos
criados por esta camada.

---

## Estrutura do repositório

```
kiro-loja-infra/
├── .github/
│   └── workflows/
│       └── validar-infra.yml        # Workflow de validação e scan (gate de qualidade)
├── foundation.yaml                  # Template CloudFormation da camada de fundação
└── kiro-loja-infra-implantar.yaml   # Arquivo de implantação do Git Sync (prod)
```

---

## Arquivos

### `foundation.yaml`

Template CloudFormation que declara toda a infraestrutura compartilhada do projeto.
É um template **reutilizável**: não tem valores de ambiente embutidos — recebe tudo via
parâmetros. Os recursos criados são:

| Recurso | Descrição |
|---|---|
| **VPC** | Rede virtual isolada com DNS habilitado (`10.0.0.0/16` por padrão) |
| **Subnets públicas** (× 2) | Uma em cada zona de disponibilidade; usadas pelo ALB |
| **Subnets privadas** (× 2) | Uma em cada zona de disponibilidade; usadas pelas tasks ECS |
| **Internet Gateway** | Permite tráfego de entrada e saída nas subnets públicas |
| **NAT Gateway** | Permite tráfego de saída das subnets privadas (pull de imagens, APIs externas) |
| **Route tables** | Pública → IGW; Privada → NAT |
| **Security Group do ALB** | Aceita HTTP (porta 80) da internet — trocar por HTTPS/443 em produção |
| **Security Group das tasks ECS** | Aceita tráfego apenas vindo do security group do ALB |
| **Application Load Balancer** | Internet-facing, nas subnets públicas; listener HTTP na porta 80 com resposta padrão 404 |
| **Cluster ECS** | Preparado para Fargate e Fargate Spot, com Container Insights habilitado |
| **ECR — pedidos** | Repositório de imagens do serviço de pedidos (scan automático no push) |
| **ECR — pagamento** | Repositório de imagens do serviço de pagamento (scan automático no push) |

Todos os recursos relevantes são exportados via `Outputs` com nomes prefixados por
`ProjectName` (ex: `kiro-loja-VpcId`, `kiro-loja-AlbListenerArn`), prontos para serem
importados pelos stacks de serviço via `Fn::ImportValue`.

---

### `kiro-loja-infra-implantar.yaml`

Arquivo de implantação lido pelo **CloudFormation Git Sync**. Não é um template
CloudFormation — não contém recursos. Sua função é informar ao Git Sync:

- **qual template implantar** (`template-file-path`)
- **com quais valores de parâmetros** (`parameters`)
- **com quais tags** aplicar a todos os recursos da stack (`tags`)

É a *configuração por ambiente*, separada do template reutilizável. Para criar um
ambiente paralelo (ex: staging), bastaria um segundo arquivo de implantação com
parâmetros distintos — o `foundation.yaml` permanece intocado.

Valores atuais:

| Campo | Valor |
|---|---|
| `template-file-path` | `foundation.yaml` |
| `ProjectName` | `kiro-loja` |
| `VpcCidr` | `10.0.0.0/16` |
| `Project` (tag) | `kiro-loja` |
| `Environment` (tag) | `prod` |

---

### `.github/workflows/validar-infra.yml`

Workflow do GitHub Actions que atua como **portão de qualidade (gate)** antes do merge
na `main`. Como o CloudFormation Git Sync implanta automaticamente qualquer mudança que
chega à `main`, este workflow garante que apenas código validado e seguro seja
implantado.

**Quando roda:** em todo Pull Request com destino à branch `main` — nos eventos de
abertura, novo commit e reabertura do PR.

**O que executa:**

| Step | Ferramenta | O que verifica |
|---|---|---|
| Lint do template | **cfn-lint** | Sintaxe YAML, tipos de propriedades, recursos inválidos, funções intrínsecas mal usadas e boas práticas AWS (erros + avisos) |
| Scan de segurança | **Checkov** | Configurações de risco: portas abertas para `0.0.0.0/0`, criptografia ausente, logs desabilitados, políticas permissivas (CIS, NIST, PCI-DSS) |

**Comportamento ao falhar:** se qualquer um dos dois steps falhar, o workflow falha e
o merge fica bloqueado. O PR só pode ser aprovado após a correção dos problemas
apontados.

**Papel no fluxo completo:**

```
PR aberto → validar-infra roda → aprovado → merge na main → Git Sync implanta
                                  ↑
                         gate obrigatório
```

Para ativar como check obrigatório, acesse:
`GitHub → Settings → Branches → Branch protection rules → main`
→ *Require status checks to pass before merging* → adicione o job **`validar`**.

---

## Deploy — Como funciona o Git Sync

O CloudFormation Git Sync monitora este repositório continuamente. O fluxo é:

1. Qualquer **push na branch `main`** que altere `foundation.yaml` ou
   `kiro-loja-infra-implantar.yaml` dispara automaticamente uma atualização da stack.
2. O CloudFormation lê o arquivo de implantação, localiza o template pelo caminho
   declarado em `template-file-path` e aplica as mudanças.
3. Se o repositório usar pull requests, o CloudFormation pode postar um resumo das
   mudanças diretamente no PR (opção *Enable comment on pull request* no console).
4. O status da sincronização e o histórico de commits aplicados ficam visíveis no
   console do CloudFormation, na aba **Git sync** da stack.

Para a configuração inicial ou para reconectar o Git Sync, acesse o console do
CloudFormation → selecione (ou crie) a stack → *Sync from Git* → aponte para este
repositório e informe `kiro-loja-infra-implantar.yaml` como o arquivo de implantação.

> **Atenção:** o Git Sync monitora apenas a branch configurada no console.
> Commits em outras branches não afetam a stack de produção.

---

## Próximos passos

- Em produção, substituir o listener HTTP (porta 80) por HTTPS (porta 443) com
  certificado ACM. O template já contém comentários indicando exatamente onde fazer
  essa troca.
- Criar stacks de serviço (ex: `kiro-loja-pedidos`, `kiro-loja-pagamento`) que
  importem os outputs desta stack via `Fn::ImportValue`.
