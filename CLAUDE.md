# MyGarage — Guia para Claude

## O que é este projeto e por que importa

**MyGarage** é um gerenciador de garagem/oficina. É o projeto portfólio definitivo do Isac para conseguir uma vaga como desenvolvedor Java. O objetivo é aprender o MÁXIMO possível sobre cada decisão técnica para conseguir explicar cada escolha em entrevistas.

### Personas atendidas (dois cenários no mesmo sistema)

O sistema mira **dois públicos com o mesmo modelo de domínio**:

1. **Usuário comum (B2C)** — a pessoa com **um único carro** que só quer gerir as manutenções do próprio veículo (lembrar de troca de óleo, não perder IPVA/seguro/revisão). Esse usuário entra pela parte de **garagem**. A complexidade de spaces/members/roles tem que ficar **invisível** pra ele: ao registrar, ganha um `Space` `GARAGE` padrão automático ("Minha Garagem") e é sempre `OWNER`. Onboarding rápido: cadastrou → adicionou carro → pronto.
2. **Usuário avançado (B2B)** — donos de várias garagens, oficinas com mecânicos, gestão de equipe com roles.

**Princípio:** o caso simples (1 space, 1 membro OWNER) é apenas um caso particular do complexo. O modelo de domínio já comporta os dois; a diferença é de **UX/defaults**, não de estrutura. Nunca obrigar o usuário de 1 carro a entender o conceito de "espaço" ou "membros".

### Mobile e PWA

O sistema precisa funcionar **plenamente em mobile** — a estratégia é **PWA (Progressive Web App)**: o site instala na tela inicial do celular e roda como app (tela cheia, ícone, offline parcial, push), sem precisar de app store. Ingredientes no front React: Web App Manifest + Service Worker + HTTPS.

**Ordem de construção: desktop-first, mas excelente no mobile.** O público que decide a contratação (recrutadores, tech leads) avalia portfólio no desktop — então a primeira impressão é construída lá. Mas o uso real do app (na garagem/oficina, celular na mão) é mobile, então a responsividade tem que ser impecável. Desktop-first é ordem de prioridade, não desculpa pra mobile capenga.

App nativo é **evolução futura** (não prioridade). Como o backend é API REST pura (contrato JSON), web, PWA e futuro app nativo consomem **a mesma API** sem mudança no backend — a separação front/back já prepara o terreno.

**Regra mais importante de todas:** nunca saia escrevendo código direto. Explique o PORQUÊ de cada decisão, mostre as alternativas existentes, dê referências de estudo. O Isac digita o código — você guia. Ele quer entender tudo, não só copiar.

Sequência de trabalho: **explicar estrutura → Isac entende → Isac cria o arquivo/código → próximo passo**.

---

## Stack definida

| Camada | Tecnologia | Por que essa escolha |
|--------|-----------|----------------------|
| Backend | Spring Boot 3.5.x, Java 21 | Padrão do mercado Java enterprise; Spring Boot 3.x exige Java 17+ |
| Segurança | Spring Security + JWT | JWT é stateless (sem sessão no servidor), escala horizontalmente, padrão para APIs REST |
| Banco | PostgreSQL 15 (Docker) | Open source, robusto, Docker para portfólio |
| ORM | Spring Data JPA (Hibernate) | JPA é abstração padrão Java; Hibernate é a implementação mais usada |
| Boilerplate | Lombok | Elimina getters/setters/construtores repetitivos sem framework extra |
| Frontend | React (projeto separado, feito depois) | Separação clara de responsabilidades; front pode ser substituído |
| IA (futuro) | Gemini Flash gratuito | 1.500 req/dia sem custo; só formatação de texto |
| Deploy | Gratuito (Railway/Render + Neon/Supabase + Vercel) | Portfólio visível online sem custo; pago ~$5/mês só quando mostrar pra recrutadores |

---

## Como o Claude deve trabalhar com o Isac

- **Nunca escreva código sem explicar o porquê antes**
- Mostre as alternativas e quando usar cada uma
- Dê referências externas de estudo (docs oficiais Spring, Baeldung, Martin Fowler)
- Diga o que fazer — o Isac digita
- Não tenha medo de ir fundo: nomenclatura, padrões de design, trade-offs
- Quando perguntar "por que X e não Y?", essa é a pergunta mais importante — responda com cuidado
- O objetivo é que o Isac consiga defender qualquer decisão em uma entrevista

---

## Organização de pacotes: Package by Feature

O projeto usa **Package by Feature** (não Package by Layer).

```
com.isacdev.mygarage/
├── auth/               # Autenticação, JWT, registro, login
│   ├── User.java       # Entidade JPA do usuário
│   ├── UserRepository.java
│   ├── AuthService.java
│   ├── AuthController.java
│   └── dto/            # LoginRequest, RegisterRequest, JwtResponse
├── space/              # Garagens e Oficinas
│   ├── Space.java      # Entidade com enum tipo (GARAGE/WORKSHOP)
│   ├── SpaceMember.java # Many-to-many com roles
│   ├── SpaceRepository.java
│   ├── SpaceService.java
│   ├── SpaceController.java
│   └── dto/
├── vehicle/            # Veículos com dados completos
│   ├── Vehicle.java
│   ├── VehicleDocument.java  # IPVA, seguro, revisão
│   ├── VehicleRepository.java
│   ├── VehicleService.java
│   ├── VehicleController.java
│   └── dto/
├── maintenance/        # Manutenção planejada + histórico
│   ├── MaintenancePlan.java   # Template: intervalo km/dias
│   ├── MaintenanceRecord.java # Registro histórico real
│   ├── MaintenanceService.java
│   ├── MaintenanceController.java
│   └── dto/
├── notification/       # Alertas de manutenção vencida/próxima
│   ├── NotificationService.java
│   └── NotificationController.java
└── config/             # Configurações globais Spring
    ├── SecurityConfig.java
    ├── JwtFilter.java
    └── CorsConfig.java
```

**Por que Package by Feature e não por Layer?**
Package by Layer (todos os controllers juntos, todos os services juntos) funciona em projetos pequenos, mas em projetos maiores você acaba abrindo 5 pastas diferentes para trabalhar numa feature. Package by Feature mantém tudo relacionado a uma feature no mesmo lugar — coesão alta, acoplamento baixo. É mais próximo do DDD (Domain-Driven Design).

**Referência:** [Package by Feature (Adam Bien)](https://www.adam-bien.com/roller/abien/entry/how_package_by_feature_not) | [DDD — Martin Fowler](https://martinfowler.com/bliki/DomainDrivenDesign.html)

---

## Modelo de Domínio

### Usuários e Autenticação

**`users`** — usuários do sistema
- `id` (UUID), `username`, `email`, `password_hash` (BCrypt), `created_at`
- Registro público (sem convite)
- Senha: BCrypt com strength 12 (padrão seguro)

### Espaços (Garagens e Oficinas)

**`spaces`** — garagens e oficinas no mesmo banco, diferenciadas por tipo
- `id` (UUID), `nome`, `tipo` (enum: `GARAGE` | `WORKSHOP`), `owner_id` (FK users), `cor_tema`, `background_url`
- Garagem e Oficina têm o mesmo modelo de dados mas o frontend aplica tema visual diferente baseado no `tipo`
- Por que usar a mesma tabela? Reduz duplicação de código; a diferença é comportamental (visual no front), não estrutural

**`space_members`** — many-to-many com roles por espaço
- `space_id`, `user_id`, `role` (enum: `OWNER` | `MANAGER` | `MECHANIC` | `VIEWER`)
- Hierarquia: OWNER → MANAGER → MECHANIC → VIEWER
  - **OWNER**: dono do espaço, pode apagar, gerenciar membros
  - **MANAGER**: adiciona/remove veículos, adiciona membros
  - **MECHANIC**: registra manutenções, atualiza km
  - **VIEWER**: só leitura
- O dono do espaço é automaticamente inserido em `space_members` com role `OWNER`

### Veículos

**`vehicles`** — dados completos do veículo
- Identificação: `marca`, `modelo`, `ano`, `placa`, `chassi` (VIN), `cor`, `apelido`
- Técnico: `km_atual`, `combustivel` (enum: GASOLINA/ETANOL/FLEX/DIESEL/ELETRICO), `cambio` (MANUAL/AUTOMATICO), `motor`
- Financeiro: `valor_compra`, `data_compra`, `valor_mercado`
- `space_id` (FK spaces)
- Foto: campo `foto_url` nullable — V1 sem foto, V2 com upload (boa história de evolução para portfólio)

**`vehicle_documents`** — documentos separados por clareza
- `vehicle_id` (FK), `vencimento_ipva`, `vencimento_seguro`, `vencimento_revisao_obrigatoria`
- Por que tabela separada? Agrupa responsabilidade; veículo pode existir sem documentos preenchidos

### Manutenção (2 tabelas distintas)

**`maintenance_plans`** — template de manutenção planejada
- `vehicle_id` (FK), `nome` ("Troca de óleo"), `descricao`
- `intervalo_km` (nullable), `intervalo_dias` (nullable) — pelo menos um deve estar preenchido
- `km_base` (km onde foi feita pela última vez), `data_base` (data onde foi feita pela última vez)
- Sistema calcula quando vai vencer: `km_base + intervalo_km` e `data_base + intervalo_dias`
- O mais próximo dos dois define o alerta

**`maintenance_records`** — histórico real de execução
- `vehicle_id` (FK), `plan_id` (FK nullable — pode ser registro avulso), `data_realizacao`, `km_no_momento`
- `custo`, `descricao`, `realizado_por` (texto livre), `notas`
- Quando um registro é criado, atualiza `km_base` e `data_base` do plano correspondente

**Por que duas tabelas e não uma?**
Plano é intenção (template). Registro é história (fato imutável). Misturar os dois numa tabela criaria campos nullable demais e lógica confusa. Separar permite auditar o histórico sem afetar o planejamento.

### Notificações

Sem tabela própria — gerado em runtime pela `NotificationService`.
Consulta `maintenance_plans` e compara com `km_atual` do veículo e data atual.
Retorna lista de alertas: vencido / próximo (menos de 7 dias ou menos de 500km).
Frontend faz request normal na abertura de cada tela — sem WebSocket por ora.

---

## Segurança

- **BCrypt** com strength 12 para senhas
- **JWT stateless**: sem sessão no servidor, token vai no header `Authorization: Bearer <token>`
- **CORS** configurado explicitamente (não `*`) — só aceita requisições do domínio do frontend
- **Validação de input** em todos os DTOs (Bean Validation: `@NotNull`, `@Size`, `@Email`)
- **Role-based access**: cada endpoint verifica o role do usuário dentro do espaço
- Nunca expor stack traces para o cliente (respostas de erro genéricas)

---

## Frontend (depois)

React puro (sem Next.js por enquanto — hosting mais simples).
Visual inspirado em Test Drive Unlimited: temático de carros, backgrounds personalizados por espaço, cores configuráveis.
Garagem tem background de garagem, oficina tem background de oficina.
O contrato entre front e back é o JSON da API — o front pode mudar visual livremente.

**PWA + responsividade:** o front deve ser empacotado como PWA (Web App Manifest + Service Worker), permitindo instalação na tela inicial sem app store. Ordem de construção **desktop-first** (público avaliador = recrutadores no desktop), mas com responsividade **impecável no mobile** (uso real do app). App nativo fica como evolução futura, consumindo a mesma API REST.

---

## Estado atual do projeto

- **Só existe `MygarageApplication.java`** — o esqueleto gerado pelo Spring Initializr
- NADA mais foi criado: nenhuma entidade, nenhum controller, nenhum repository
- O Spring Initializr só gera o esqueleto — toda a lógica precisa ser criada manualmente
- `application.properties` está vazio — precisa de configuração do banco

### Próximos passos (em ordem)

1. Criar `docker-compose.yml` para subir o PostgreSQL local
2. Configurar `src/main/resources/application.properties` (URL do banco, credenciais, JPA)
3. Adicionar dependências JWT ao `pom.xml`
4. Criar pacote `auth/` com `User.java` (primeira entidade JPA)
5. Criar `UserRepository.java` (interface Spring Data JPA)
6. Criar `AuthService.java` + `AuthController.java`
7. Configurar Spring Security (`SecurityConfig.java`)
8. Implementar JWT (`JwtFilter.java`, `JwtService.java`)

---

## Dependências JWT para adicionar ao pom.xml (quando chegar lá)

```xml
<!-- JJWT — JSON Web Token -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>

<!-- Validação de input (Bean Validation) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

---

## Comandos Docker

```bash
docker compose up -d               # subir PostgreSQL
docker compose down                # parar (preserva dados)
docker compose down -v             # parar E apagar dados
docker logs mygarage_db -f         # ver logs do banco
docker exec -it mygarage_db psql -U isac -d mygarage  # conectar ao banco
```

---

## Referências de estudo

| Tópico | Link |
|--------|------|
| Spring Boot oficial | https://docs.spring.io/spring-boot/docs/current/reference/html/ |
| Spring Security | https://docs.spring.io/spring-security/reference/ |
| Spring Data JPA | https://docs.spring.io/spring-data/jpa/reference/ |
| JWT explicado | https://jwt.io/introduction |
| JPA entities e relacionamentos | https://www.baeldung.com/jpa-entities |
| BCrypt e senhas | https://www.baeldung.com/spring-security-registration-password-encoding-bcrypt |
| Package by Feature | https://www.adam-bien.com/roller/abien/entry/how_package_by_feature_not |
| DDD com Spring | https://www.baeldung.com/hexagonal-architecture-ddd-spring |
| Lombok guia | https://projectlombok.org/features/ |
| REST API design | https://restfulapi.net/ |

---

## Decisões pendentes / features futuras

- **Fotos de veículos**: V1 sem foto (campo `foto_url` nullable), V2 com upload
- **Ilustração automática**: buscar imagem por marca/modelo/ano via API externa (NHTSA/CarQuery)
- **IA (Gemini Flash)**: formatação de texto nos registros de manutenção — implementar depois do core
- **Notificações por email**: fora do escopo inicial; in-app suficiente

---

## Agent skills

### Issue tracker

Issues rastreadas no GitHub (`IsacCostaDev/mygarage`), via CLI `gh`. See `docs/agents/issue-tracker.md`.

### Triage labels

Labels padrão (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Contexto único — `CONTEXT.md` + `docs/adr/` na raiz do repo. See `docs/agents/domain.md`.
