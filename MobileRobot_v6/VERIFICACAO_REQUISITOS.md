# Verificação de requisitos — MobileRobot v6

## Executar no OSATE

1. Atualize o projeto com **F5** e salve os editores.
2. Abra **Window → Show View → Other… → Alisa → Assurance View**.
3. Selecione **MobileRobotV6_AllRequirements** e execute **Verify All** pelo menu de contexto.
4. Expanda os cenários e os subsistemas para consultar cada requisito.

O caso completo contempla o cenário nominal e pico. O caso `MobileRobotV6_RAMCase` continua disponível apenas para os dois testes de orçamento de RAM por cenário.
A execução pelo ALISA avalia instâncias do modelo; não lê os relatórios CSV. Para refazer as demais análises, reinstancie o modelo atualizado.

## O que é verificado

| Grupo | Tratamento |
| --- | --- |
| 12 deadlines | Comparação automática do máximo de `SEI::Response_Time` com o prazo do requisito e com a propriedade `Deadline`. Também exige resposta positiva e não inferior ao WCET. |
| RAM estimada da Raspberry | Soma dos orçamentos, limitada a 80% de `RAMCapacity`. |
| RAM estimada da ESP32 | Soma dos orçamentos, limitada à capacidade de RAM de dados definida. |
| Contratos de interface | Revisão manual e evidência de integração pendentes. Compatibilidade de tipos não comprova o comportamento. |
| `RAM_Reserve` original | Pendente de evidência sobre RAM utilizável e consumo de pico. Não é substituído pela estimativa de orçamento. |
| `RAM_Runtime` original | Pendente do ensaio completo de 30 minutos. |

Resultado esperado com os valores atuais: **28 aprovados e 32 pendentes (soma dos dois cenários)**.
São 14 verificações automáticas e 16 atividades manuais pendentes por cenário.

Os itens manuais devem permanecer pendentes até existir evidência; não os aprove somente para deixar o caso verde. `Verify All` não executa o ensaio físico nem produz essa evidência.

## Origem dos limites de resposta

Os limites são analíticos e condicionais ao modelo atual: tarefas fixadas nos núcleos, prioridade fixa preemptiva (maior número = maior prioridade no modelo), períodos/intervalos mínimos declarados e ausência de interferência adicional não modelada. Não são medições do Zephyr/chonOS.

Foi utilizada a recorrência `R = C + B + soma(ceil(R/Tj) * Cj)` para as tarefas de maior prioridade no mesmo núcleo. A IMU usa 3,5 ms de CPU mais 1,53 ms de espera I²C, tratados conservadoramente como 5,03 ms antes da interferência do controle. Filas anteriores ao despacho não fazem parte desses prazos.

| Tarefa | Resposta máxima (ms) | Deadline (ms) |
| --- | ---: | ---: |
| control_task | 5 | 10 |
| imu_task | 15,03 | 20 |
| battery_task | 1,3 | 500 |
| telemetry_task | 1,1 | 20 |
| rx_task | 0,6 | 20 |
| command_guard_task | 0,1 | 10 |
| bridge_task | 0,8 | 20 |
| odom_task | 1,4 | 20 |
| nav2_task | 15 | 50 |
| slam_task | 60 | 100 |
| integration_task | 5 | 200 |
| jason_task | 85 | 500 |

**O `.verify` compara os limites anotados; não recalcula o escalonamento.** Ao mudar WCET, períodos, prioridades, bindings, bloqueios ou tarefas, refaça a análise e atualize `SEI::Response_Time`. O teste de escalonamento por CPU não inclui automaticamente a espera I²C. As propriedades adicionadas também podem ser utilizadas pela análise de latência; gere novamente seus relatórios.

## Arquivos

- `robot_requirements.reqspec`: requisitos existentes; deadlines passam a ler o limite de resposta anotado.
- `robot_requirements.verify`: um claim para cada requisito original.
- `robot_evidence.methodregistry`: atividades manuais para os requisitos que ainda exigem evidência.
- `robot_requirements.alisa`: caso completo.
- `robot_model_ram.reqspec`, `robot_ram.verify`, `robot_ram.alisa`: testes separados do orçamento de RAM.
- `esp32_software.aadl` e `raspberry_software.aadl`: limites de resposta que faltavam; funções, períodos, prioridades e WCET preservados.

Referência: [guia oficial de requisitos e verificação do ALISA](https://github.com/osate/alisa-examples/blob/master/Documentation/BasicRequirementSpecificationGuidance.md).
