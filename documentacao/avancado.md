# ☕🌱 | Avançado
Foco: Segurança (Security), Testes Automatizados, Performance (Async/Cache) e Observabilidade.

## 1. Segurança e Autenticação (Spring Security)🛡️
> Contexto: Segurança — Uso: Obrigatório em Produção

O Spring Security é um framework poderoso de autenticação e autorização.

### O Padrão Stateless (JWT)
Em APIs REST modernas, evitamos sessões no servidor (Stateful). Preferimos Tokens JWT (Json Web Tokens).

1. Usuário loga.
2. Servidor gera um Token assinado.
3. Cliente envia esse Token no Header (Authorization: Bearer <token>) em cada requisição.

### Anotações Principais 🏷️

| Anotação                   | Significado / Função                                              | Quando usar                                                                 |
|----------------------------|-------------------------------------------------------------------|-----------------------------------------------------------------------------|
| ***@EnableWebSecurity***         | Habilita a configuração customizada de segurança.                 | Na classe de configuração de segurança (ex: `SecurityConfig`).              |
| ***@PreAuthorize***              | Restringe o acesso a um método com base em roles/permissões.       | Em `Controller` ou `Service`. Ex: `@PreAuthorize("hasRole('ADMIN')")`.       |
| ***@AuthenticationPrincipal***   | Injeta o usuário autenticado diretamente no método.               | Quando é necessário saber quem está fazendo a requisição sem nova consulta ao banco. |

***Exemplo de Configuração Moderna (Security Filter Chain)***
Em versões recentes (Spring Boot 3+), não herdamos mais de WebSecurityConfigurerAdapter. Usa-se Beans:
````java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable()) // Desabilitar CSRF para APIs Stateless
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.POST, "/login").permitAll() // Libera login
            .requestMatchers("/admin/**").hasRole("ADMIN") // Protege área admin
            .anyRequest().authenticated() // Bloqueia o resto
        )
        .addFilterBefore(meuJwtFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
````
---

## 2. Testes Automatizados (Testing) 🧪
> Contexto: Qualidade de Código — Uso: Profissional

O Spring Boot facilita testes de integração que sobem o contexto da aplicação (simulam o servidor rodando).

| Anotação           | Contexto                  | Função                                                                 | Diferença Chave                                                                 |
|--------------------|---------------------------|------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| ***@SpringBootTest***    | Teste de Integração       | Carrega todo o contexto da aplicação (banco, configurações e beans).   | Mais lento, porém testa o fluxo real completo da aplicação.                      |
| ***@WebMvcTest***        | Teste de Fatia (Slice)    | Carrega apenas a camada Web (Controllers).                              | Rápido; não carrega `Service` nem `Repository`.                                  |
| ***@MockBean***          | Mocking (Simulação)       | Cria um mock de um Bean e o injeta no contexto do Spring.               | Essencial para isolar camadas (ex: testar o Controller simulando o Service).     |
| ***@ActiveProfiles***    | Configuração              | Define qual perfil será utilizado durante o teste (ex: `"test"`).      | Útil para usar banco H2 em memória ou configurações específicas de teste.        |

**Cenário Típico de Teste de Controller:**
````java
@WebMvcTest(UsuarioController.class)
class UsuarioControllerTest {

    @Autowired MockMvc mockMvc; // Simula chamadas HTTP
    @MockBean UsuarioService service; // Simula a lógica de negócio

    @Test
    void deveRetornarSucesso() throws Exception {
        // Arrange (Preparação)
        when(service.buscarPorId(1L)).thenReturn(new UsuarioDTO(...));

        // Act & Assert (Ação e Verificação)
        mockMvc.perform(get("/usuarios/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.nome").value("Teste"));
    }
}
````

---
## 3. Performance e Assincronismo⚡
> Contexto: Otimização — Uso: Cenários de Alta Carga

Não bloqueie a thread principal do usuário com tarefas lentas (envio de e-mail, geração de relatórios pesados).

### `@Async` e `@EnableAsync`
- 🧠 **Função** ➜ Executa o método em uma thread separada (background). O Controller responde imediatamente ao usuário enquanto o processo roda no fundo.
- 🧪 **Requisito** ➜ Adicionar @EnableAsync na classe main/config.
- ✅ **Boas Práticas** ➜ Métodos @Async não devem retornar valores diretamente (use CompletableFuture ou void).

### `@Cacheable` e `@EnableCaching`
- 🧠 **Função** ➜ Armazena o retorno de um método em cache (Redis, memória, etc.). Na próxima chamada com os mesmos parâmetros, o método não é executado; o valor é retornado do cache.

**Exemplo:**
````java
@Cacheable("produtos") // Nome do cache
public List<Produto> listarTodos() {
    return repository.findAll(); // Só executa se não estiver no cache
}
````
> ⚠️ *Atenção: Lembre-se de usar @CacheEvict para limpar o cache quando os dados forem atualizados.*

### `@Scheduled`
- 🧠 **Função** ➜ Executa métodos automaticamente em intervalos definidos (Cron Jobs).
- 🔨 **Uso** ➜ Relatórios noturnos, limpeza de banco de dados.

Exemplo: `@Scheduled(cron = "0 0 0 * * ?")` (Meia-noite todo dia).

---

## 4. Gerenciamento de Ambientes (Profiles) 👤
> Contexto: DevOps — Uso: Essencial

Nunca use configurações de Produção em Desenvolvimento.

### `@Profile`
- 🧠 **Função** ➜ Indica que um Bean ou Configuração só deve ser carregado em um perfil específico.
- 🔨 **Uso** ➜ Ter um Bean de envio de e-mail real para "prod" e um Bean que apenas loga no console para "dev".

- **Configuração via Properties** ~ Crie arquivos separados:

`application-dev.properties` (Banco local, logs verbose)

`application-prod.properties` (Banco nuvem, logs error)

No `application.properties` principal, ative: `spring.profiles.active=dev`

---

## 5. Observabilidade (Actuator) 👀
> Contexto: Operações/SRE — Uso: Monitoramento

Como saber se sua aplicação está viva e saudável em produção?

- 🧩 **Dependência** ➜ `spring-boot-starter-actuator`
- 🪄 **Endpoints Mágicos** ➜ O Spring expõe URLs nativas para monitoramento.
  - `/actuator/health`: Status da aplicação (UP/DOWN) e de dependências (Banco, Disk Space).
  - `/actuator/metrics`: Métricas detalhadas (uso de memória, CPU, requisições HTTP).
  - `/actuator/loggers`: Permite mudar o nível de log (DEBUG/INFO) em tempo de execução sem reiniciar o app.

--- 
### Resumo visual geral 🧠

Imagine sua aplicação Spring Boot como uma cebola em camadas.🧅
Aqui está onde cada parte do nosso guia se encaixa:

- **Núcleo (Infra)** ⟶ *ApplicationContext, Profiles, Actuator*.
- **Dados (Repository)** ⟶ *JPA, Hibernate, Transactions*.
- **Lógica (Service)** ⟶ `@Service`, *Async, Caching, Regras de Negócio*.
- **Interface (Web)** ⟶ *RestController, DTOs, Validation, ExceptionHandling*.
- **Borda (Segurança)** ⟶ *Spring Security, JWT Filter, CORS.*
