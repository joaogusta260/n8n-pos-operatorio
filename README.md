# n8n-pós-operátorio
Fluxo do n8n para automatizar atendimento de pós-operatório de clínica de cirurgia plástica.

## Visão Geral do Fluxo

Na imagem abaixo, é possível ver que o fluxo está dividido em **quatro grandes seções** (cada cor representa um bloco lógico):
![Fluxo do n8n](./ia%20pos%20operatorio.png)


### (Bordô) – Entrada e Normalização dos Dados
- Um **Webhook** (`input evolution`) recebe as mensagens do WhatsApp.  
- Nós de “if” e “set” verificam se a mensagem é saída ou entrada (fromMe/incoming) e fazem a normalização dos campos (como `message.message_id`, `content_type`, etc.).  
- **Redis** é usado para “pausar” ou “retomar” um atendimento (*timeout*), garantindo que não seja enviado um grande volume de mensagens de forma sequencial.

### (Roxo) – Concatenação de Mensagens
- Os nós “push message buffer” e “get messages buffer” armazenam os textos recebidos numa lista (Redis).  
- É feito um “delete buffer” quando é detectado que a mensagem já pode ser concatenada.  
- Em seguida, o nó `messages` junta todas as mensagens em um só campo, facilitando a análise pela IA.

### (Azul) – Filtragem das Mensagens
- O fluxo verifica se a mensagem é de **texto**, **áudio**, **imagem** ou **documento**.  
- Dependendo do tipo, realiza diferentes ações:
  - Transcrição de áudio (OpenAI → Mensagem de Áudio → Converter Áudio)  
  - Análise de imagem (OpenAI → Converter Imagem)  
  - Extração de texto de documentos (PDFs, etc.)  
- Após a filtragem, o campo final é formatado para que a IA possa processar o conteúdo.

### (Verde) – IA e Ferramentas
- **OpenAI Chat** (via “AI Agent”) recebe o texto processado e utiliza instruções muito específicas para fornecer respostas humanizadas e acolhedoras (especialmente para pacientes de pós-operatório).  
- A memória (**Redis Chat Memory**) mantém o contexto de conversas passadas, garantindo mais coerência nos diálogos.  
- Conforme o tipo de sintoma detectado, o fluxo pode acionar ferramentas que geram tarefas em paralelo, como `Entrar_em_contato`, `Enviar_equipe` ou `Perguntas_apos_sintomas`.

### (Envio da Resposta)
- Por fim, o nó “Enviar Mensagem WhatsApp Evo 2.1.0” dispara a resposta automática de volta para o paciente, usando as orientações finais da IA.  
- Este fluxo está **ativo em produção** e atende atualmente pacientes em período de pós-operatório, agilizando o atendimento e reduzindo o tempo demandado pelo médico.

---

## Destaques Técnicos

### n8n
- Orquestra todo o fluxo de automação.  
- Permite verificar tipo de conteúdo (texto, áudio, imagem, PDF) e condicionar passos.

### Redis
- Utilizado para “bufferizar” mensagens e gerenciar sessão de chat/memória.  
- Gerencia também o *timeout* de atendimento, evitando que mensagens sejam enviadas sem espaçamento.

### OpenAI (LangChain)
- Transcreve áudio para texto (no caso de mensagens de voz).  
- Analisa imagem (básico) para descrição ou triagem.  
- Gera texto de resposta usando instruções específicas de tom de voz e regras do pós-operatório.

### Ferramentas (Tool Workflows)
- **Entrar_em_contato** e **Enviar_equipe**: Chamadas para sub-fluxos no n8n que enviam notificações extras para a equipe médica.  
- **Perguntas_apos_sintomas**: Fluxo adicional que aprofunda a conversa quando detectados sintomas como cefaleia ou enjoo.

### HTTP Requests
- Integrações com API do WhatsApp (envio de mensagens).  
- APIs internas para conversão de mídia (áudio → base64, imagem → base64, etc.).

---

## Benefícios

- **Redução de tempo** do médico, que não precisa responder cada paciente manualmente.  
- **Atendimento 24/7**: a IA responde de forma imediata, filtrando casos urgentes.  
- **Registro estruturado** de todo o histórico via Redis / logs do n8n.  
- **Escalável**: pode ser adaptado para outras clínicas ou fluxos de atendimento.
