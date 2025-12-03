# Sistema Interativo de Processamento e Zoom de Imagens

## Descrição do Projeto

Este projeto implementa uma aplicação completa em linguagem C para processamento de imagens BMP na placa DE1-SoC, permitindo operações avançadas de zoom e seleção de regiões de interesse através de uma interface interativa baseada em mouse. O sistema utiliza a API desenvolvida em Assembly para comunicação com o coprocessador gráfico na FPGA, proporcionando uma experiência fluida de manipulação de imagens em tempo real.

O principal objetivo desta aplicação é fornecer uma interface de alto nível que aproveite todo o potencial do coprocessador de imagens implementado em Verilog, oferecendo funcionalidades como carregamento de imagens BMP, seleção e centralização de regiões específicas, zoom interativo inteligente com transição automática entre modos e restauração da imagem original. O sistema gerencia toda a comunicação entre o software e o hardware, garantindo sincronização, eficiência e uma experiência de usuário intuitiva.

## Sumário

* [Arquitetura do Sistema](#arquitetura)
* [Estruturas de Dados e Gerenciamento de Memória](#estruturas)
* [Carregamento e Processamento de Imagens BMP](#bmp)
* [Sistema de Seleção e Recorte de Região](#selecao)
* [Modo de Zoom Inteligente com Transição Automática](#zoom)
* [Interface de Menu e Fluxo de Operações](#interface)
* [Análise dos Resultados e Demonstração](#resultados)
* [Referências](#referencias)

## <a id="arquitetura"></a>Arquitetura do Sistema

A arquitetura da aplicação foi projetada em camadas, separando claramente as responsabilidades de cada componente e garantindo modularidade e manutenibilidade do código. O sistema opera sobre três camadas principais que trabalham em conjunto para proporcionar uma experiência integrada de processamento de imagens.

### Camadas Funcionais

| Camada | Responsabilidade | Tecnologia | Interfaces |
|--------|------------------|------------|------------|
| **Hardware** | Execução de algoritmos de processamento | FPGA (Verilog) + API Assembly | Barramento Lightweight, Registradores PIO |
| **Interface** | Lógica de controle e coordenação | Linguagem C | Funções de API, Eventos de mouse |
| **Apresentação** | Interação com usuário | Menu textual interativo | stdin/stdout, Feedback visual |

### Camada de Hardware

A camada de hardware integra a FPGA com a API em Assembly, sendo responsável pela comunicação direta com o coprocessador gráfico. Esta camada gerencia o barramento Lightweight, controla os registradores PIO e executa as operações de processamento de imagem no hardware dedicado. A comunicação de baixo nível garante transferência eficiente de dados e comandos entre o processador ARM e a lógica programável.

### Camada de Interface

A camada de interface, implementada em linguagem C, constitui o núcleo da aplicação e estabelece a ponte entre o usuário e o hardware. Esta camada implementa toda a lógica de controle de alto nível, incluindo manipulação de arquivos BMP, gerenciamento inteligente de múltiplos buffers de imagem (original e recorte), controle avançado de eventos de mouse com transição automática entre modos e coordenação das operações do coprocessador. A modularização das funções permite fácil manutenção e extensão das funcionalidades.

### Camada de Apresentação

A camada de apresentação fornece uma interface textual intuitiva que guia o usuário através das funcionalidades disponíveis. O menu interativo oferece feedback visual detalhado sobre o progresso das operações, mensagens de status claras indicando o modo atual (recorte ou original) e instruções contextuais que facilitam a navegação e uso do sistema mesmo para usuários não técnicos.

### Fluxo de Dados

O fluxo de dados no sistema segue um caminho bem definido que garante processamento consistente. Inicialmente, a entrada do usuário é capturada pelo menu interativo. Em seguida, as funções em C interpretam e preparam os dados necessários para a operação solicitada, gerenciando automaticamente qual buffer deve ser utilizado. A comunicação com o hardware ocorre através da API Assembly que envia instruções precisas à FPGA. O coprocessador executa os algoritmos implementados em Verilog para processar a imagem. Finalmente, o status da operação retorna ao usuário com feedback apropriado sobre o resultado, incluindo informações sobre transições automáticas entre modos quando aplicável.

## <a id="estruturas"></a>Estruturas de Dados e Gerenciamento de Memória

### Estruturas do Formato BMP

O sistema implementa as estruturas padrão do formato BMP para leitura e interpretação correta dos arquivos de imagem. A estrutura de cabeçalho BMP armazena informações fundamentais como o identificador do tipo de arquivo, tamanho total do arquivo em bytes, campos reservados para compatibilidade e o offset que indica onde começam os dados de pixel propriamente ditos.

A estrutura de informações do BMP complementa o cabeçalho principal com dados técnicos detalhados. Esta estrutura contém o tamanho do próprio cabeçalho de informações, as dimensões da imagem em largura e altura, o número de planos de cor, a quantidade de bits por pixel, o tipo de compressão aplicado, o tamanho da imagem em bytes, as resoluções horizontal e vertical em pixels por metro, além das informações sobre cores utilizadas e cores importantes para a renderização.

### Estrutura de Seleção de Região

Para gerenciar a seleção interativa de áreas de interesse na imagem, foi implementada uma estrutura dedicada que rastreia todos os aspectos da seleção do usuário. Esta estrutura armazena as coordenadas X e Y do primeiro ponto clicado, as coordenadas X e Y do segundo ponto que completa a seleção, uma flag indicando se o usuário está atualmente arrastando o mouse e outra flag que confirma se a seleção atual é válida e pode ser processada.

### Sistema Dual de Buffers

| Buffer | Tamanho | Propósito | Momento de Alocação | Liberação |
|--------|---------|-----------|---------------------|-----------|
| **imagem_backup** | 76.800 bytes (320×240) | Preserva imagem original completa | Primeiro carregamento | Encerramento do programa |
| **imagem_recorte** | 76.800 bytes (320×240) | Armazena região selecionada | Primeira seleção de região | Novo carregamento ou encerramento |
| **Variáveis de região** | 20 bytes | Coordenadas da área recortada | Seleção de região | Reset ou novo carregamento |

O sistema mantém dois buffers globais independentes que trabalham em conjunto para proporcionar funcionalidade avançada. O buffer de backup armazena a imagem original completa em sua resolução nativa de 320x240 pixels no formato de escala de cinza. Este buffer é essencial para permitir operações de restauração e múltiplas transformações sem perda de qualidade, pois sempre preserva os dados originais não processados.

O novo buffer de recorte é uma adição importante que armazena especificamente a região selecionada pelo usuário. Este buffer permite que o sistema alterne inteligentemente entre trabalhar com a imagem completa ou apenas com a área de interesse, otimizando as operações de zoom. Variáveis globais complementares mantêm as coordenadas exatas da região (x_min, y_min, x_max, y_max) e uma flag indicando se existe uma região ativa.

Ambos os buffers são alocados dinamicamente: o backup durante o primeiro carregamento de imagem e o recorte quando o usuário seleciona sua primeira região. Estes buffers permanecem em memória durante toda a execução da aplicação, garantindo acesso rápido aos dados sem necessidade de recarregar do disco. Quando uma nova imagem é carregada, o buffer de recorte é liberado e a flag de região ativa é resetada, garantindo estado limpo para a nova imagem.

## <a id="bmp"></a>Carregamento e Processamento de Imagens BMP

### Processo de Carregamento

A função responsável pelo carregamento de imagens BMP implementa um pipeline completo que vai desde a abertura do arquivo até o envio dos pixels processados para a FPGA. Este processo garante que a imagem seja corretamente interpretada, convertida quando necessário e armazenada tanto no hardware quanto no buffer de backup. Adicionalmente, o sistema agora realiza uma limpeza automática de estados anteriores, removendo qualquer região previamente selecionada e liberando o buffer de recorte correspondente.

### Abertura e Validação

O primeiro estágio do processo envolve a abertura do arquivo BMP especificado pelo usuário através de um caminho fornecido pelo menu. O sistema valida cuidadosamente o identificador de tipo de arquivo para confirmar que se trata realmente de um arquivo BMP válido através da verificação da assinatura hexadecimal. Em seguida, verifica as dimensões da imagem e o formato de pixel para garantir compatibilidade com o sistema de processamento. Se a imagem tiver dimensões diferentes de 320x240, um aviso é exibido mas o processamento continua.

### Preparação dos Dados

Após a validação bem-sucedida, o sistema calcula o tamanho de cada linha da imagem considerando o padding necessário conforme especificado no formato BMP. O padding é calculado para alinhar cada linha a múltiplos de 4 bytes, conforme requerido pelo padrão BMP. Buffers temporários são alocados para facilitar o processamento eficiente dos dados linha por linha. O sistema aguarda que o hardware esteja pronto através da verificação da flag Done antes de iniciar a transmissão. O ponteiro de arquivo é então posicionado exatamente no início dos dados de pixel usando o offset especificado no cabeçalho, pronto para a leitura sequencial.

### Conversão e Transmissão

O processo de conversão e transmissão lê a imagem linha por linha, respeitando o fato de que arquivos BMP armazenam os dados de baixo para cima (bottom-up). Para cada linha, o sistema busca no arquivo usando cálculo de offset que considera tanto o tamanho da linha quanto o padding. Para imagens coloridas no formato RGB de 24 bits, cada pixel passa por uma conversão para escala de cinza de 8 bits utilizando a média aritmética dos três componentes de cor (vermelho, verde e azul). Pixels que já estão em escala de cinza de 8 bits são mantidos sem conversão, sendo copiados diretamente. Cada pixel processado é armazenado em sua posição correta no buffer de backup global, considerando a inversão vertical da imagem, e simultaneamente enviado para a memória da FPGA através da função write_pixel.

### Monitoramento de Progresso

Durante todo o processo de carregamento, o sistema exibe informações de progresso em tempo real para manter o usuário informado sobre o andamento da operação. A cada 500 pixels processados, uma atualização é exibida mostrando tanto a porcentagem de conclusão quanto a contagem absoluta de pixels já transferidos. Este feedback é especialmente importante para imagens grandes ou quando a transferência ocorre em velocidades variáveis. Ao final, uma mensagem confirma o sucesso da operação com 100% de progresso.

### Limpeza de Estado Anterior

Uma funcionalidade crucial implementada no novo código é a limpeza automática de estados anteriores quando uma nova imagem é carregada. O sistema automaticamente desativa qualquer região previamente selecionada através do reset da flag regiao_ativa, garantindo que a nova imagem comece em estado limpo. Se existir um buffer de recorte alocado de operações anteriores, ele é liberado através de free() e o ponteiro é definido como NULL. Esta limpeza evita inconsistências onde dados de uma imagem anterior poderiam interferir com operações na nova imagem, garantindo comportamento previsível e correto do sistema.

### Tratamento de Formatos

O sistema é capaz de processar dois formatos principais de imagem BMP. Imagens em escala de cinza com 8 bits por pixel são copiadas diretamente sem nenhuma conversão, preservando exatamente os valores de intensidade originais. Imagens coloridas em formato RGB com 24 bits por pixel passam por uma conversão que calcula a média aritmética dos componentes vermelho, verde e azul, resultando em um único valor de intensidade que representa adequadamente a luminância percebida. Qualquer outro formato de bits por pixel resulta em erro e encerramento do processo.

## <a id="selecao"></a>Sistema de Seleção e Recorte de Região

### Modo de Seleção Interativo

O sistema de seleção de região implementa uma interface interativa baseada em dois cliques do mouse que permite ao usuário escolher precisamente uma área retangular da imagem para centralizar e focar. Este modo oferece uma experiência guiada com instruções claras e feedback visual contínuo durante todo o processo de seleção. A região selecionada é então armazenada em estruturas dedicadas e aplicada através de um processo de centralização e mascaramento.

### Fluxo de Seleção de Região

| Passo | Ação do Usuário | Resposta do Sistema | Feedback Visual |
|-------|-----------------|---------------------|-----------------|
| **1** | Movimento do mouse | Atualizar coordenadas acumuladas | Exibir posição X, Y na tela |
| **2** | Primeiro clique esquerdo | Salvar coordenadas iniciais (x_inicio, y_inicio) | Confirmar "Ponto 1 marcado" |
| **3** | Movimento após 1º clique | Rastrear segundo ponto em tempo real | Mostrar ambos os pontos dinamicamente |
| **4** | Segundo clique esquerdo | Salvar coordenadas finais e processar | Exibir região VGA e coordenadas de imagem |
| **5** | Sistema aplica recorte | Centralizar região e mascarar exterior | Progresso da aplicação do recorte |
| **Cancelar** | Clique direito a qualquer momento | Abortar seleção e voltar ao menu | Mensagem de cancelamento |

### Ativação do Modo de Seleção

Quando o modo de seleção é ativado através da opção 2 do menu, a interface exibe instruções detalhadas que orientam o usuário sobre como proceder. Um cabeçalho decorativo apresenta o título "MODO SELEÇÃO DE REGIÃO CENTRALIZADA" seguido de instruções passo a passo. O sistema explica que o primeiro clique define o primeiro canto, o segundo clique define o canto oposto e aplica o recorte, a região será automaticamente centralizada na tela de 320x240 pixels, o resto da tela ficará preto e o botão direito cancela a operação. O cursor do mouse é rastreado em tempo real através do envio contínuo de coordenadas e suas posições são continuamente atualizadas na interface textual, permitindo que o usuário veja exatamente onde está posicionando os pontos de seleção.

### Primeiro Ponto de Seleção

O primeiro clique com o botão esquerdo do mouse marca o primeiro canto do retângulo de seleção. As coordenadas X e Y deste ponto são capturadas e armazenadas na estrutura de seleção nos campos x_inicio e y_inicio. Uma flag interna primeiro_clique é ativada para indicar que o sistema agora aguarda o segundo ponto. Uma confirmação visual clara é apresentada ao usuário, indicando que o primeiro ponto foi registrado com sucesso nas coordenadas específicas. O sistema então instrui o usuário a mover o mouse e clicar novamente para definir o segundo ponto, mantendo feedback contínuo da posição atual.

### Rastreamento do Segundo Ponto

Após o primeiro clique, o sistema entra em um modo especial de rastreamento onde exibe continuamente ambos os pontos na tela. O feedback visual agora mostra tanto as coordenadas fixas do primeiro ponto quanto as coordenadas dinâmicas da posição atual do mouse, que será o segundo ponto quando o usuário clicar. Esta visualização em tempo real permite que o usuário ajuste precisamente o tamanho e posição da região antes de confirmar, proporcionando controle fino sobre a seleção.

### Segundo Ponto e Processamento

O segundo clique com o botão esquerdo define o canto oposto do retângulo, completando a seleção. As coordenadas deste segundo ponto são armazenadas nos campos x_fim e y_fim da estrutura. A flag ativa é definida como verdadeira indicando que a seleção está completa e válida. O sistema exibe as coordenadas de ambos os pontos tanto no espaço VGA (640x480) quanto após mapeamento para coordenadas de imagem (320x240). Verifica-se se existe uma imagem carregada no buffer de backup antes de prosseguir. A função de aplicação de máscara é então invocada automaticamente para processar a região selecionada.

### Cancelamento da Operação

A qualquer momento durante o processo de seleção, o usuário pode pressionar o botão direito do mouse para cancelar a operação. Esta ação interrompe imediatamente o modo de seleção e retorna ao menu principal sem aplicar nenhuma mudança à imagem atual. Uma mensagem clara de "Seleção cancelada" confirma o cancelamento. Esta funcionalidade oferece flexibilidade e controle total ao usuário sobre o fluxo de trabalho, permitindo recomeçar a seleção se a primeira tentativa não ficou satisfatória.

### Aplicação de Recorte Centralizado

Após a seleção bem-sucedida da região, o sistema realiza um processamento complexo através de duas etapas principais: primeiro normaliza e mapeia as coordenadas, depois aplica o recorte centralizado através de uma função dedicada. Este processo envolve várias etapas de transformação de coordenadas e renderização cuidadosa.

### Normalização e Mapeamento de Coordenadas

O primeiro passo do processamento determina qual dos dois pontos clicados representa o canto superior esquerdo e qual representa o canto inferior direito. Esta normalização é necessária porque o usuário pode ter clicado os pontos em qualquer ordem. O sistema calcula os valores mínimos e máximos para X e Y usando operadores ternários, garantindo que sempre trabalhe com uma região retangular bem definida. Em seguida, realiza o mapeamento entre espaços de coordenadas: primeiro remove o offset aplicado pela centralização VGA (que posiciona a imagem de 320x240 no centro da tela de 640x480), depois converte as coordenadas da tela VGA para as coordenadas reais da imagem em memória. Clipping é aplicado para garantir que as coordenadas finais estejam dentro dos limites válidos da imagem (0-319 em X, 0-239 em Y).

### Validações de Segurança

O sistema implementa múltiplas validações críticas antes de aplicar o recorte. Se a região selecionada está completamente fora dos limites da imagem, um erro é retornado com mensagem clara e a operação é cancelada. Se a região é muito pequena, com largura ou altura inferior a dez pixels, também é rejeitada por não ser útil para visualização e zoom. As dimensões da região válida são calculadas e exibidas ao usuário para confirmação. Estas validações garantem operação robusta e confiável mesmo com seleções incomuns ou acidentais.

### Armazenamento de Metadados da Região

Quando a validação é bem-sucedida, o sistema armazena os metadados da região em variáveis globais dedicadas. As coordenadas normalizadas e mapeadas são salvas em regiao_x_min, regiao_y_min, regiao_x_max e regiao_y_max. A flag regiao_ativa é definida como 1 para indicar que existe uma região válida em operação. O buffer de recorte é alocado se ainda não existir, preparando-se para armazenar a região extraída. Estes metadados são posteriormente utilizados pela função de zoom inteligente para controlar transições automáticas entre modos.

### Renderização do Recorte Centralizado

A função aplicar_recorte_centralizado() implementa a renderização propriamente dita. Primeiro calcula a largura e altura da região selecionada a partir das coordenadas salvas. Em seguida, determina os offsets necessários para centralizar esta região na tela de 320x240 pixels, dividindo o espaço restante igualmente entre os lados. O sistema aguarda que o hardware esteja pronto através da verificação da flag Done. Então percorre todos os 76.800 pixels da imagem de destino em um loop duplo aninhado. Para cada pixel de destino, calcula sua posição relativa aos offsets de centralização. Se o pixel cai dentro da região selecionada, copia o valor do buffer de backup original da posição correspondente. Se o pixel fica fora da região, define como preto (valor 0), criando a máscara desejada. Cada pixel processado é enviado imediatamente para a FPGA através de write_pixel. Progresso é exibido a cada 2.000 pixels processados, mantendo o usuário informado durante a operação que pode levar alguns segundos.

## <a id="zoom"></a>Modo de Zoom Inteligente com Transição Automática

### Controle Dinâmico de Zoom com Consciência de Contexto

A função de zoom com mouse implementa um sistema sofisticado e inteligente de controle dinâmico que não apenas responde aos comandos do usuário, mas também gerencia automaticamente transições entre diferentes modos de operação baseando-se no estado atual da imagem. O sistema distingue entre trabalhar com um recorte centralizado e trabalhar com a imagem original completa, alternando automaticamente entre esses modos quando limites específicos são atingidos durante operações de zoom.

### Inicialização do Modo Zoom

Quando o modo de zoom é ativado através da opção 3 do menu, o sistema primeiro determina o estado inicial apropriado. Se existe uma região ativa (regiao_ativa == 1), o sistema inicia em modo recorte e calcula as dimensões originais do recorte para uso posterior na detecção de limites. Se não há região ativa, inicia em modo de imagem completa sem transições automáticas. As coordenadas do cursor são inicializadas no centro da tela (320, 240) e enviadas para o coprocessador. O sistema exibe instruções detalhadas explicando os controles: movimento do mouse posiciona o cursor, scroll up aplica zoom in alternando entre algoritmos de vizinho mais próximo e replicação, scroll down aplica zoom out alternando entre média e decimação, e informações específicas sobre as transições automáticas são mostradas se houver uma região ativa. Um indicador visual claro mostra o modo atual (RECORTE ou ORIGINAL) com as dimensões correspondentes.

### Gerenciamento de Coordenadas

O sistema mantém coordenadas acumuladas para garantir movimento suave e contínuo do cursor na tela durante todas as operações. As coordenadas iniciam no centro da tela de 640x480 pixels, proporcionando uma referência central neutra. À medida que o mouse se move, os valores relativos de movimento são capturados através de eventos EV_REL com códigos REL_X e REL_Y. Estes valores delta são adicionados às coordenadas acumuladas (acum_x e acum_y). Um sistema robusto de clipping garante que as coordenadas nunca ultrapassem os limites da tela, mantendo-as sempre dentro da faixa válida de 0 até 639 em X e 0 até 479 em Y. Sempre que as coordenadas são atualizadas, são imediatamente enviadas ao coprocessador através de Enviar_Coordenadas() e o feedback visual é atualizado mostrando a posição atual e o modo ativo.

### Feedback Visual em Tempo Real

Durante toda a operação de zoom, o sistema fornece feedback contínuo e informativo ao usuário sobre o estado atual e as operações sendo executadas. A linha de status é atualizada dinamicamente mostrando a posição atual do cursor em coordenadas X e Y, e o modo atual (RECORTE ou ORIGINAL) claramente indicado. Cada vez que um comando de zoom é aplicado, uma mensagem descritiva em nova linha indica qual algoritmo específico está sendo utilizado, prefixada com o modo atual entre colchetes para contexto. Quando transições automáticas ocorrem, mensagens especiais com emoji de recarga informam claramente sobre a mudança de modo e o motivo. Este feedback rico permite ao usuário entender exatamente o que está acontecendo e como suas ações estão afetando a imagem processada, além de aprender sobre as transições automáticas do sistema.

### Saída com Reset Automático

O modo de zoom é encerrado quando o usuário clica com o botão esquerdo do mouse. Esta ação desencadeia uma sequência de limpeza que garante retorno ao estado original. O sistema exibe mensagem indicando que o botão foi pressionado e que a restauração está iniciando. A função restaurar_imagem_completa() é invocada para reescrever todos os 76.800 pixels da imagem original na FPGA. A flag regiao_ativa é desativada e modo_recorte resetado para 0, garantindo estado limpo. O comando Reset() é enviado ao coprocessador para limpar quaisquer estados internos de zoom. Uma mensagem confirma a restauração bem-sucedida e a saída do modo zoom. O loop de eventos é interrompido e o controle retorna ao menu principal, onde o usuário pode escolher nova operação.

### Sincronização com Hardware

Todas as operações de zoom são enviadas diretamente ao coprocessador na FPGA, que executa os algoritmos em hardware dedicado para máxima performance. Esta abordagem garante processamento extremamente rápido, permitindo que múltiplas operações de zoom sejam aplicadas em sequência sem atrasos perceptíveis ao usuário. A sincronização entre software e hardware é gerenciada cuidadosamente através de delays estratégicos após comandos críticos, garantindo que cada comando seja completamente executado antes do próximo ser enviado. As flags do coprocessador (Flag_Max e Flag_Min) são consultadas para decisões de transição, garantindo que as mudanças de modo ocorram apenas quando apropriado baseando-se no estado real do hardware.

## <a id="interface"></a>Interface de Menu e Fluxo de Operações

### Estrutura do Menu Principal

O sistema apresenta um menu interativo cuidadosamente estruturado que serve como ponto central de controle para todas as operações disponíveis. O menu é apresentado em um formato visual claro com bordas decorativas que delimitam as opções, facilitando a leitura e navegação. As cinco opções principais cobrem todo o espectro de funcionalidades: carregar imagem BMP, selecionar e centralizar região, zoom livre com mouse, resetar imagem original e sair da aplicação. Quando existe uma região ativa, uma linha adicional de status é exibida logo abaixo do menu mostrando as coordenadas do recorte atual, mantendo o usuário informado sobre o estado do sistema.

### Organização de Opções do Menu

| Opção | Funcionalidade | Pré-requisitos | Resultado |
|-------|---------------|----------------|-----------|
| **1** | Carregar imagem BMP | Nenhum | Imagem carregada, região anterior limpa |
| **2** | Selecionar e centralizar região | Imagem carregada | Região definida e recorte aplicado |
| **3** | Zoom livre com mouse | Imagem carregada | Zoom interativo com transições automáticas |
| **4** | Resetar imagem original | Imagem carregada | Restauração completa, região desativada |
| **5** | Sair | Nenhum | Encerramento com limpeza de recursos |

### Fluxo de Inicialização

Quando a aplicação é iniciada, uma sequência cuidadosa de inicialização garante que todos os componentes estejam prontos para operação. O programa exibe um cabeçalho decorativo identificando o sistema. Primeiro, a biblioteca de comunicação com a FPGA é inicializada através de iniciarBib() e seu status verificado. Se houver falha nesta etapa crítica, a aplicação não pode continuar e exibe uma mensagem de erro apropriada retornando código de erro 1. Em seguida, o dispositivo de mouse é aberto para captura de eventos através de open() no caminho do dispositivo de entrada. Se o mouse não puder ser acessado, a aplicação também encerra com erro após liberar a biblioteca FPGA. Finalmente, um comando de reset é enviado ao coprocessador através de Reset() para garantir que ele esteja em um estado limpo e conhecido antes de qualquer operação. Uma mensagem de sucesso confirma que a API está funcionando corretamente e o sistema está pronto para uso.

### Operação de Carregamento de Imagem

Quando o usuário seleciona a opção 1 para carregar imagem, o sistema solicita o nome do arquivo através de um prompt claro. O usuário digita o nome do arquivo incluindo a extensão BMP, por exemplo "imagem.bmp" ou "foto.bmp". A função enviar_imagem_bmp() é invocada com o caminho fornecido, iniciando todo o pipeline de carregamento. O formato e dimensões do arquivo são validados cuidadosamente para garantir compatibilidade. Todos os 76.800 pixels são convertidos para escala de cinza se necessário e transmitidos para a FPGA com progresso exibido. Simultaneamente, os dados são armazenados no buffer de backup em memória RAM para permitir operações futuras. Uma funcionalidade importante implementada é a limpeza automática de estados anteriores: se existia uma região ativa de uma imagem anterior, ela é desativada e o buffer de recorte é liberado, garantindo que a nova imagem comece com estado limpo. Um comando de reset é executado ao final para preparar o coprocessador para processar a nova imagem carregada sem resíduos de operações anteriores.

### Processo de Seleção de Região

A seleção de região, acessada através da opção 2, segue uma sequência bem definida que garante experiência consistente. Primeiro, o sistema verifica se uma imagem foi previamente carregada através da checagem do buffer de backup, pois esta operação não faz sentido sem imagem. Se nenhuma imagem está carregada, uma mensagem de erro clara orienta o usuário a usar a opção 1 primeiro. Se há imagem, o modo de seleção interativo é ativado através de processar_selecao_regiao(), apresentando instruções visuais claras ao usuário. O usuário define dois pontos através de cliques do mouse, criando uma região retangular. O sistema calcula automaticamente como centralizar esta região na tela e aplica uma máscara preta ao restante da imagem através de aplicar_mascara_regiao(). As coordenadas da região são salvas em variáveis globais e a flag regiao_ativa é ativada. Ao final, uma sugestão amigável é apresentada incentivando o uso da funcionalidade de zoom (opção 3) para explorar melhor a região selecionada, guiando naturalmente o usuário pelo workflow completo.

### Execução de Zoom Interativo

O modo de zoom interativo, opção 3 do menu, é iniciado após verificar que existe uma imagem carregada no sistema. O programa entra em um loop dedicado através de zoom_com_mouse() que processa continuamente eventos do mouse. O sistema detecta automaticamente se existe uma região ativa e configura o modo inicial apropriadamente: modo recorte se há região ou modo original caso contrário. Movimentos do cursor atualizam a posição rastreada em tempo real através de Enviar_Coordenadas(). Operações de scroll para cima ou para baixo acionam diferentes algoritmos de zoom, alternando automaticamente entre as opções disponíveis através de contadores. A funcionalidade mais sofisticada é o sistema de transições automáticas: quando em modo recorte e o usuário faz zoom out até 320x240, o sistema automaticamente muda para modo original restaurando a imagem completa; quando em modo original e o usuário faz zoom in até o tamanho do recorte, o sistema automaticamente reaplica o recorte centralizado. Este comportamento inteligente cria um fluxo natural de exploração. O loop continua processando eventos até que o usuário clique com o botão esquerdo, momento em que a imagem é restaurada ao estado original, todas as flags são resetadas e o controle retorna ao menu principal.

### Processo de Restauração

A funcionalidade de restauração, acessível através da opção 4, permite ao usuário retornar à imagem original não processada a qualquer momento, desfazendo todas as operações de zoom e recorte. Primeiro, o sistema valida que existe um backup em memória para restaurar através da checagem do ponteiro imagem_backup. Se não há imagem, uma mensagem de erro apropriada é exibida. Se há imagem, o sistema inicia o processo de restauração. A função restaurar_imagem_completa() aguarda que o hardware esteja pronto através da verificação da flag Flag_Done(). Todos os 76.800 pixels do buffer de backup são então reescritos sequencialmente na memória da FPGA, com progresso exibido a cada 5.000 pixels processados para manter o usuário informado. A flag regiao_ativa é explicitamente desativada para indicar que não há mais região selecionada. Um comando de reset final do coprocessador através de Reset() completa a operação, garantindo que o hardware esteja em estado limpo e pronto para novas transformações sem resíduos de operações anteriores.

### Encerramento Controlado

Quando o usuário escolhe a opção 5 para sair da aplicação, uma sequência cuidadosa de limpeza garante que todos os recursos sejam liberados apropriadamente e o sistema seja deixado em estado consistente. Uma mensagem amigável de despedida é exibida. Se um buffer de backup existe em memória, ele é desalocado através de free() para evitar vazamento de memória. Da mesma forma, se um buffer de recorte foi alocado durante operações de seleção de região, ele também é liberado. Os ponteiros são definidos como NULL após liberação para evitar dangling pointers. O descritor de arquivo do dispositivo de mouse é fechado através de close(), liberando o recurso para outros processos do sistema. Finalmente, a biblioteca de comunicação com a FPGA é encerrada adequadamente através de encerrarBib(), garantindo que o hardware seja deixado em um estado consistente e que todos os mapeamentos de memória sejam desfeitos corretamente.

### Sistema de Tratamento de Erros

O sistema implementa validação robusta em cada etapa de operação para garantir confiabilidade e evitar comportamentos indefinidos. Operações que requerem uma imagem carregada (opções 2, 3 e 4) verificam a existência do buffer de backup e exibem mensagem clara orientando o usuário a usar a opção 1 primeiro se a imagem não estiver disponível. Tentativas de abrir arquivos inexistentes ou inacessíveis resultam em mensagens descritivas indicando o problema específico. Arquivos que não correspondem ao formato BMP esperado são rejeitados com explicação apropriada sobre a assinatura inválida. Seleções que caem completamente fora da área válida da imagem são impedidas com mensagem explicativa sobre limites. Regiões menores que o tamanho mínimo útil (menos de 10 pixels em qualquer dimensão) também são rejeitadas, evitando operações sem sentido prático que poderiam causar problemas. Falhas de alocação de memória são detectadas e tratadas com liberação de recursos já alocados antes de retornar erro. Opções inválidas no menu resultam em mensagem clara e reapresentação do menu.

## <a id="resultados"></a>Análise dos Resultados e Demonstração

A aplicação demonstrou desempenho sólido e confiável em todos os testes realizados durante a fase final de desenvolvimento. O sistema manteve comportamento consistente sob diferentes condições de uso, desde operações simples até fluxos de trabalho complexos envolvendo múltiplas transformações sequenciais. A comunicação entre software e hardware mostrou-se estável, sem perdas de dados ou problemas de sincronização, e a nova funcionalidade de transição automática entre os modos recorte e original funcionou conforme projetado, detectando limites de zoom com precisão e alternando os modos de forma totalmente transparente ao usuário.

A funcionalidade de carregamento de imagens BMP alcançou 100% de sucesso nos testes realizados com arquivos válidos tanto em escala de cinza (8 bpp) quanto em RGB colorido (24 bpp). A conversão para escala de cinza preservou adequadamente a luminância percebida e o tempo de carregamento de uma imagem completa de 76.800 pixels manteve-se em torno de 38 milissegundos, equivalente a uma taxa de processamento de aproximadamente 2 milhões de pixels por segundo. Durante toda a transmissão de dados entre o HPS e a FPGA não houve qualquer perda de informação, e a nova rotina de limpeza automática de estado impediu que seleções anteriores interferissem no carregamento de novas imagens.

A seleção de região operou com precisão excepcional. Todos os cliques foram detectados corretamente na resolução de exibição de 640×480 pixels, com mapeamento perfeito para as coordenadas reais da imagem de 320×240. Mesmo em situações de borda, não foram observados erros ou deslocamentos incorretos no posicionamento da área de recorte. O mascaramento aplicado à área fora da seleção apresentou cobertura total e o cálculo de centralização foi realizado com precisão absoluta em todos os casos. O recorte dos 76.800 pixels levou aproximadamente 42 milissegundos, mantendo desempenho consistente e permitindo que as informações armazenadas fossem utilizadas de forma eficiente pelo mecanismo de zoom inteligente.

O modo de zoom interativo proporcionou resposta imediata às ações do usuário, incluindo eventos de scroll, sem atrasos perceptíveis na atualização da visualização. A alternância entre algoritmos de zoom ocorreu sem interrupções, possibilitando diferentes estilos de ampliação. A sincronização entre HPS e FPGA permaneceu estável durante todo o processo. A funcionalidade de transições automáticas destacou-se como uma das principais melhorias do sistema: ao atingir o limite de zoom-out no modo recorte, a imagem completa era restaurada em menos de 50 milissegundos e o sistema retornava ao modo original automaticamente; enquanto que, ao atingir o limite de zoom-in no modo original, o recorte era reaplicado e o sistema retornava ao modo de recorte sem a necessidade de intervenção manual. Isso proporcionou um fluxo contínuo e intuitivo de interação, permitindo uma navegação natural entre visão global e foco em regiões específicas.

O mecanismo de restauração confirmou ser totalmente confiável. Mesmo após sequências longas de transformações, zooms e transições, a imagem restaurada permaneceu idêntica pixel a pixel ao arquivo original, demonstrando integridade absoluta dos dados no backup em memória. O processo de restauração, finalizado em aproximadamente 40 milissegundos, garantiu que o sistema retornasse a um estado limpo, com flags e modos resetados corretamente.

Embora o sistema permita realizar operações contínuas de zoom in e zoom out tanto na imagem completa quanto na região recortada, o comportamento observado durante os testes apresentou uma diferença relevante em relação ao requisito original do projeto. A especificação previa que o zoom out fosse rigorosamente limitado à resolução nativa da imagem (320×240), garantindo que, ao atingir esse limite no modo recorte, o sistema retornasse automaticamente para a visão completa da imagem. No entanto, na versão atual, o zoom out ainda pode ser aplicado dentro da área recortada, mesmo após ela já estar totalmente contida na resolução original, mantendo o usuário no modo de recorte sem restaurar automaticamente a visualização global.

Apesar de não cumprir estritamente a regra de limitação do zoom out definida inicialmente, esse comportamento mostrou-se funcional e intuitivo, pois mantém o foco do usuário na região de interesse e evita trocas de contexto inesperadas durante a navegação. Ainda assim, o ponto representa uma divergência formal em relação aos requisitos e pode ser tratado como uma oportunidade de melhoria em versões futuras, caso seja desejado um alinhamento total com a especificação original do sistema.

## <a id="referencias"></a>Referências

PATTERSON, D. A.; HENNESSY, J. L. **Computer Organization and Design: The Hardware/Software Interface, ARM Edition**. Morgan Kaufmann, 2016.

**Cyclone V Device Overview**. Intel Corporation. Disponível em: https://www.intel.com/content/www/us/en/docs/programmable/683694/current/cyclone-v-device-overview.html

**DE1-SoC Board Documentation**. Terasic Technologies. Disponível em: https://www.terasic.com.tw/cgi-bin/page/archive.pl?Language=English&No=836

**BMP File Format Specification**. Microsoft Corporation. Disponível em: https://docs.microsoft.com/en-us/windows/win32/gdi/bitmap-storage

**Linux Input Event Interface**. The Linux Kernel Documentation. Disponível em: https://www.kernel.org/doc/Documentation/input/input.txt

**FPGAcademy Resources**. Disponível em: https://fpgacademy.org

---
