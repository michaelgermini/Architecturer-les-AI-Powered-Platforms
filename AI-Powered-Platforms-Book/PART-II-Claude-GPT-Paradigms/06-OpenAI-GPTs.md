# 6. OpenAI GPTs : Function Calling & Autonomy

*Agents personnalisés et logique d'exécution dynamique*

---

## 🎯 Le paradigme GPTs

### Qu'est-ce qu'un GPT ?

> **GPT (Generative Pre-trained Transformer)** : Un modèle de langage avancé capable de comprendre le contexte, raisonner étape par étape, et exécuter des actions via des appels de fonctions.

Contrairement aux Skills modulaires, les GPTs privilégient **l'autonomie et l'adaptabilité**.

### Architecture d'un GPT Agent

```python
class GPTAgent:
    """Architecture de base d'un GPT Agent"""

    def __init__(self, config: dict):
        self.model = config['model']  # gpt-4, gpt-3.5-turbo
        self.tools = config['tools']  # Liste des fonctions disponibles
        self.system_prompt = config['system_prompt']
        self.memory = ConversationBuffer(window_size=10)
        self.max_iterations = config.get('max_iterations', 5)

    def execute(self, user_request: str) -> AgentResult:
        """Exécution d'une requête utilisateur"""
        # Initialisation du contexte
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": user_request}
        ]

        # Boucle de raisonnement
        for iteration in range(self.max_iterations):
            # Génération de réponse
            response = self.call_gpt(messages)

            # Vérification d'appel de fonction
            if self.has_function_call(response):
                # Exécution de la fonction
                function_result = self.execute_function_call(response)

                # Ajout au contexte
                messages.append(response)
                messages.append({
                    "role": "function",
                    "name": response['function_call']['name'],
                    "content": json.dumps(function_result)
                })
            else:
                # Réponse finale
                return AgentResult(
                    success=True,
                    response=response['content'],
                    metadata={
                        'iterations': iteration + 1,
                        'tools_used': self.get_tools_used(messages)
                    }
                )

        # Timeout
        return AgentResult(
            success=False,
            error="Max iterations reached",
            metadata={'iterations': self.max_iterations}
        )
```

---

## 🔧 Function Calling : Le mécanisme clé

### Comment ça marche

#### 1. Définition des fonctions
```python
# Définition des outils disponibles
tools = [
    {
        "type": "function",
        "function": {
            "name": "run_terminal_command",
            "description": "Execute a terminal command",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "The command to execute"},
                    "working_directory": {"type": "string", "description": "Working directory"}
                },
                "required": ["command"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read content from a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "file_path": {"type": "string", "description": "Path to the file"},
                    "max_lines": {"type": "integer", "description": "Maximum lines to read"}
                },
                "required": ["file_path"]
            }
        }
    }
]
```

#### 2. Prompt système intelligent
```python
system_prompt = """
You are an expert DevOps engineer helping with development tasks.

You have access to these tools:
- run_terminal_command: Execute terminal commands
- read_file: Read file contents
- search_codebase: Search for code patterns
- deploy_application: Deploy to staging/production

When a user asks for something:
1. Think step-by-step about what needs to be done
2. Use the appropriate tools to gather information
3. Execute the necessary actions
4. Provide clear explanations of what you're doing

Be proactive but ask for clarification if something is unclear.
"""
```

#### 3. Exécution dynamique
```python
def execute_function_call(self, response):
    """Exécution d'un appel de fonction"""
    function_call = response['function_call']
    function_name = function_call['name']
    function_args = json.loads(function_call['arguments'])

    # Routage vers la bonne fonction
    if function_name == "run_terminal_command":
        return self.run_terminal_command(**function_args)
    elif function_name == "read_file":
        return self.read_file(**function_args)
    # ... autres fonctions

    raise ValueError(f"Unknown function: {function_name}")
```

---

## 🎨 Types d'agents GPT

### 1. Specialized Agents (Agents spécialisés)

#### DevOps Agent
```python
class DevOpsAgent(GPTAgent):
    """Agent spécialisé dans les opérations DevOps"""

    def __init__(self):
        tools = [
            "run_terminal_command",
            "check_service_status",
            "scale_deployment",
            "analyze_logs",
            "rollback_deployment"
        ]
        super().__init__(
            model="gpt-4",
            tools=tools,
            system_prompt=self.get_devops_prompt()
        )

    def get_devops_prompt(self):
        return """
        You are a senior DevOps engineer responsible for keeping systems running smoothly.

        Your responsibilities include:
        - Monitoring system health
        - Troubleshooting issues
        - Performing deployments
        - Scaling services
        - Analyzing performance metrics

        Always prioritize system stability and user impact.
        """
```

#### Code Review Agent
```python
class CodeReviewAgent(GPTAgent):
    """Agent spécialisé dans la review de code"""

    def __init__(self):
        tools = [
            "read_file",
            "run_tests",
            "check_security",
            "analyze_complexity",
            "suggest_improvements"
        ]
        super().__init__(
            model="gpt-4",
            tools=tools,
            system_prompt=self.get_review_prompt()
        )

    def get_review_prompt(self):
        return """
        You are an expert code reviewer with deep knowledge of software engineering best practices.

        When reviewing code, consider:
        - Code correctness and logic
        - Security vulnerabilities
        - Performance implications
        - Code maintainability
        - Adherence to coding standards

        Provide constructive, specific feedback.
        """
```

### 2. General Purpose Agents (Agents généralistes)

#### Developer Assistant
```python
class DeveloperAssistant(GPTAgent):
    """Assistant développeur généraliste"""

    def __init__(self):
        # Large palette d'outils
        tools = [
            "code_generation",
            "debugging_help",
            "documentation_writing",
            "testing_assistance",
            "architecture_advice",
            "performance_optimization"
        ]
        super().__init__(
            model="gpt-4-turbo",
            tools=tools,
            system_prompt=self.get_assistant_prompt()
        )

    def get_assistant_prompt(self):
        return """
        You are an AI programming assistant helping developers write better code.

        You can help with:
        - Writing and explaining code
        - Debugging issues
        - Code reviews
        - Architecture decisions
        - Best practices
        - Learning new technologies

        Be helpful, patient, and encouraging.
        """
```

---

## 🧠 Avantages de l'approche Agent

### Autonomie et adaptabilité

#### Reasoning en contexte
```python
# L'agent peut raisonner étape par étape
conversation_flow = [
    {"role": "user", "content": "My app is running slow in production"},
    {"role": "assistant", "content": "Let me investigate this step by step..."},
    {"role": "assistant", "content": "First, I should check the current metrics"},
    # Function call: check_metrics
    {"role": "function", "name": "check_metrics", "content": "{...}"},
    {"role": "assistant", "content": "I see high CPU usage. Let me check the logs"},
    # Function call: analyze_logs
    {"role": "function", "name": "analyze_logs", "content": "{...}"},
    # Continue reasoning...
]
```

#### Apprentissage conversationnel
- **Mémoire contextuelle** : L'agent se souvient de la conversation
- **Adaptation progressive** : Amélioration des réponses au fil du temps
- **Personnalisation** : Apprentissage des préférences utilisateur

### User Experience fluide
- **Interface conversationnelle** : Pas besoin d'interfaces complexes
- **Guidage intelligent** : L'agent pose les bonnes questions
- **Exécution automatique** : Actions réalisées sans intervention manuelle

---

## ⚖️ Comparaison Skills vs Agents

### Skills : Précision et contrôle
- ✅ **Prédictibilité** : Comportement déterministe
- ✅ **Testabilité** : Tests isolés possibles
- ✅ **Sécurité** : Isolation des contextes
- ✅ **Performance** : Optimisation spécialisée

### Agents : Flexibilité et autonomie
- ✅ **Adaptabilité** : Gestion de cas imprévus
- ✅ **User Experience** : Interface naturelle
- ✅ **Apprentissage** : Amélioration continue
- ✅ **Complexité** : Gestion de tâches composites

---

## ✅ Checklist : Implémentation d'un GPT Agent

### Configuration ✅
- [ ] Modèle approprié sélectionné
- [ ] Tools définis et testés
- [ ] System prompt clair
- [ ] Limites d'itération définies

### Fonctionnalités ✅
- [ ] Function calling opérationnel
- [ ] Error handling robuste
- [ ] Conversation memory
- [ ] Response validation

### Sécurité ✅
- [ ] Input sanitization
- [ ] Function access control
- [ ] Rate limiting
- [ ] Audit logging

---

## 🚀 Vers la comparaison détaillée

Dans le prochain chapitre, nous comparerons en détail les approches **Skills vs Agents** et leurs cas d'usage optimaux.
