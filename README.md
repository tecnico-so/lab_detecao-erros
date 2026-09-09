# Guião sobre ferramentas de deteção de erros

![IST](img/IST_DEI.png)

## Objetivos

No final deste guião, deverá ser capaz de:

- Compreender a utilização básica do `gdb` na identificação e análise dos erros do programa;
- Utilizar o mecanismo de pontos de paragem do `gdb` para analisar erros do programa;
- Utilizar sanitizadores para analisar comportamentos incorretos do programa em tempo de execução.


## Introdução

Um programa compilar sem erros não significa que esteja correto. 
Durante a execução podem surgir acessos inválidos à memória ou valores inesperados. 
Nestes casos, saber que o programa falhou é apenas o primeiro passo: é necessário descobrir **onde**, **como** e **porque** ocorreu o erro.

Neste laboratório irá aprender a investigar estes problemas com ferramentas utilizadas no desenvolvimento de software em C. 
Vamos começar por um *debugger* para observar o estado de um programa quando ocorre uma falha e identificar o percurso de execução que a provocou. 

Depois, vamos ver os sanitizadores de código, que permitem detetar automaticamente determinadas classes de erros durante a execução, emitindo avisos.

O domínio destas ferramentas será cada vez mais importante, à medida que os programas se tornam mais complexos e os erros mais difíceis de diagnosticar.

Um programa compilar sem erros não significa que esteja correto. 
Durante a execução podem ocorrer acessos inválidos à memória ou surgir valores inesperados. 
Nestes casos, saber que o programa falhou é apenas o primeiro passo: é necessário descobrir **onde**, **como** e **por que razão** ocorreu o erro.

Neste laboratório vamos aprender a investigar estes problemas com ferramentas usadas no desenvolvimento de software em C. 
Começaremos por um *debugger*, que permite observar o estado do programa quando ocorre uma falha e reconstruir o percurso de execução que a originou.
Depois, veremos os sanitizadores de código, que detetam automaticamente determinadas classes de erros durante a execução e emitem avisos para diagnóstico.

O domínio destas ferramentas será cada vez mais importante à medida que os programas se tornam mais complexos e os erros mais difíceis de diagnosticar.

### Antes de começar

Vai precisar de um sistema operativo Ubuntu Linux ou outro.
Se ainda não o tiver disponível no seu computador pessoal, pode utilizar um dos computadores do laboratório.

Algumas das ferramentas, como o `gdb` poderão não estar instaladas.
Deve utilizar o gestor de pacotes da sua distribuição Linux para instalar as ferramentas.

Para obter os exemplos de código, clone este repositório, usando o comando: git clone https://github.com/tecnico-so/lab_detecao-erros.git


## 1. Depurador `gdb`

O `gdb` (abreviatura de *GNU Debugger*) permite analisar o que está a acontecer dentro de um programa enquanto este está em execução ou o estado de um programa antes de este terminar abruptamente.
A documentação completa da ferramenta de depuração `gdb` pode ser consultada em: <http://www.gnu.org/software/gdb/documentation>.

Para demonstrar as capacidades do `gdb` serão usados os ficheiros contidos na pasta `src` do repositório.

1. Adicione a seguinte função no início do ficheiro [`test.c`](./src/test.c).

```c
   void list_tree(node_t* p)
   {
      if (p->left)
         list_tree(p->left);
      printf("%ld\n", p->key);
      if (p->right)
         list_tree(p->right);
   }
```

2. Modifique a função `main` para oferecer o novo comando `l` que irá listar a árvore recorrendo à nova função `list_tree`.

3. Gere o programa test recorrendo à `Makefile` fornecida.

```sh
   make
```

4. Execute o programa `test` e, no estado inicial (árvore vazia), introduza o novo comando.
O que acontece?

```sh
   ./test
   > l
```

O facto do programa ter falhado com uma falta de segmentação (*segmentation fault*) foi uma surpresa?
Quando este tipo de erros acontece, nem sempre é óbvio identificar a causa, ou seja, o *bug* que esteve na origem da falha.
Nestes casos, o `gdb` pode ser uma ferramenta muito útil.
As seguintes alíneas mostram como.

*Nota: Para usar o gdb nas alíneas seguintes, é necessário que o seu programa seja compilado com a opção `-g`.*
*Pode verificar nas `CFLAGS` da `Makefile` que isto já está a acontecer quando executa o comando `make`.*

5. Execute de novo o programa `test`, mas agora sob o controlo do `gdb`:

```sh
   gdb test
```

6. Dentro do `gdb`, inicie a execução do programa com o comando `run`, ou simplesmente `r`.  
*Nota: caso quisesse passar argumentos de linha de comando ao programa, poderia especificá-los como argumentos do comando `run`.*

```sh
   (gdb) r
```

7. Tal como antes, introduza o comando `l` para o programa listar o conteúdo da árvore (vazia).
Como esperado, o programa irá de novo falhar com *segmentation fault*.
No entanto, desta vez o `gdb` está a controlar a execução e apresentará a seguinte mensagem:

```sh
   Program received signal SIGSEGV, Segmentation fault.
   0x000055555555525d in list_tree (p=0x0) at test.c:8
   8 if (p->left)
```

8. Neste momento, é como se a falta estivesse congelada, o que permite ao programador analisar o estado da execução do programa nesse exato momento para entender o que causou a falha.
Para saber qual o caminho percorrido pelo programa até chegar a esse ponto, use o comando `backtrace` (abreviado `bt`), o qual irá mostrar informação semelhante à seguinte.

```sh
   (gdb) bt
   #0 0x000055555555525d in list_tree (p=0x0) at test.c:8
   #1 0x0000555555555444 in main () at test.c:47
   (gdb)
```

O primeiro número em cada linha indica o nível em que essa função está, começando pela função onde o programa terminou abruptamente (neste caso, `list_tree`).
Observe como o `gdb` mostra os argumentos passados às funções ao longo da pilha (neste caso, apenas o argumento `p`, sendo indicado o seu valor).

*Nota: É frequente serem mostradas funções que são de sistema (embora não seja este o caso).*
*Quando isso se verifica, obviamente, o que interessa é a última função que correu do nosso programa, pois muito provavelmente será aí que está o erro.*

9. Com esta informação, já detetou o *bug*?
Então saia do `gdb` (use o comando `quit` ou `q`) e corrija o _bug_ no ficheiro `test.c`.

## 2. Pontos de paragem (*breakpoints*)

No exemplo anterior, analisou o estado de um programa no momento em que este falhou (com *segmentation fault*).
Outra forma útil de entender o comportamento do programa é usando *breakpoints*.
Por exemplo, quando se acrescenta uma nova função a um programa já existente, é usual colocar-se um *breakpoint* para acompanhar, passo a passo, a forma como esse novo código se comporta.
De seguida ilustramos usando a nova função `list_tree`, assumindo que já corrigiu o *bug* que detetou nos passos anteriores.

1. De novo, carregue o programa `test` no `gdb`:

```sh
   gdb test
```

2. Utilize o comando `break` (abreviado `b`) para colocar um _breakpoint_ na primeira instrução da nova função, `list_tree`:

```sh
   (gdb) b list_tree
```

3. Para confirmar que o *breakpoint* foi definido, pode pedir para listar os pontos de paragem da seguinte forma:

```sh
   (gdb) info b
```

*Nota: Um breakpoint pode ser disabled, enabled ou apagado usando, respetivamente, os comandos `disable n`, `enable n` e `delete n`, em que **n** representa o número do breakpoint indicado pelo comando `info b`.*

4. Execute o programa usando o comando `run` (abreviado `r`):

```sh
   (gdb) r
```

5. A aplicação é executada normalmente ficando a aguardar *input*.
Introduza algumas entradas na árvore e peça para listar o conteúdo da mesma, por exemplo assim:

```sh
   > a 20
   > a 15
   > a 30
   > l
```

6. Quando o programa chega a um *breakpoint* é interrompido pelo `gdb`, aparecendo no ecrã a linha de código onde o programa parou.
Pode ver em detalhe o código onde se encontra utilizando o comando `list` (abreviado `l`):

```sh
   (gdb) l
```

7. Acompanhe, passo a passo, a execução da função à medida que a árvore é recursivamente listada, usando as seguintes dicas:

- Pode avançar pelo código, linha a linha, utilizando o comando `next` (abreviado `n`);
- Sempre que a execução chega a uma linha em que é chamada uma função (por exemplo, quando `list_tree` é chamada recursivamente para uma das sub-árvores), pode decidir entrar dentro dessa chamada usando o comando `step` (abreviado `s`), em vez do `next`;
- Em cada momento, pode pedir para avaliar qualquer expressão, por exemplo, valores de variáveis no contexto atual, usando o comando `print <expressão>` (abreviado `p`).  
*Nota: não pode observar variáveis que ainda não foram definidas, tendo de esperar até estar na linha seguinte à da definição para poder inspecioná-las*;
- Pode mudar de contexto com o comando `frame <nº de frame>` (abreviado `f`), em que o número do _frame_ é o primeiro número mostrado pelo comando `backtrace` antes de cada contexto.
Isto é útil para observar variáveis definidas noutros contextos;
- Caso se tenha esquecido de definir _breakpoints_, pode pausar a execução do programa com `Ctrl-C`.
Esta combinação de teclas normalmente termina o programa em execução, mas dentro do `gdb` é interpretada como pausa.

A [documentação oficial do ´gdb´](https://sourceware.org/gdb/current/onlinedocs/gdb/) é extensa, mas existem *cheat sheets*, que são [referências sumárias dos comandos mais usados no `gdb`](https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf).

## 3. Sanitizadores de código

O `gcc` (abreviatura de *GNU C Compiler*) permite que o programa a compilar seja instrumentado com rotinas que verificam, em tempo de execução, se determinados comportamentos incorretos ocorrem.
Estas opções de instrumentação chamam-se sanitizadores de código (*code sanitizers*).
Existem diferentes tipos de sanitizadores ([lista de sanitizadores do GCC](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html)), cada um focado em detetar diferentes classes de erros.

Note que:

- Nem todos os sanitizadores podem ser combinados num mesmo programa compilado;
- O programa tem de ser **todo** compilado com o mesmo conjunto de sanitizadores.

Entre outros, os seguintes sanitizadores pode ser especialmente úteis para os programas C que desenvolveremos em SO: *AddressSanitizer*, *UndefinedBehaviorSanitizer*, *ThreadSanitizer* (este apenas quando, mais tarde, aprendermos a fazer programas concorrentes).
Que tipo de erros deteta cada um destes sanitizadores?
Consulte na documentação do `gcc` (listada acima).

1. Para usar cada sanitizador, basta usar a opção `-fsanitize` do `gcc` aquando da compilação do seu programa.
Veja na documentação referida acima como usar esta opção para ativar cada tipo de sanitizador.

2. Experimente agora voltar ao programa que compôs na alínea 1.1, que tinha um *bug*.
Edite a `Makefile` para, nas `CFLAGS`, incluir um ou mais sanitizadores de código à sua escolha.

3. Compile o programa (provavelmente terá de fazer `make clean`) e experimente de novo correr o programa.
Se adicionou o sanitizador certo, então o seu programa será interrompido assim que o sanitizador deteta o *bug* e verá uma mensagem de erro que o ajudará a corrigir o problema.

*Nota: Outros compiladores como o `clang` também têm os seus sanitizadores.*
*Atenção que estas operações variam ligeiramente entre diferentes compiladores.*

## Ferramenta de Depuração Visual: `ddd`

Os depuradores mais modernos incluem também uma componente de representação visual do estado da memória do programa.
A ferramenta `ddd` (*Data Display Debugger*) foi uma das primeiras ferramentas a adotar esta abordagem e que pode ser usada como interface gráfica para o `gdb`.

Nesta [imagem ilustrativa](https://www.gnu.org/software/ddd/all.png) podemos ver como as variáveis e os ponteiros são representados.

O [manual do `ddd`](https://www.gnu.org/software/ddd/manual/) começa com um [exemplo prático](https://www.gnu.org/software/ddd/manual/html_mono/ddd.html#Sample-Session) da utilização da ferramenta.


## Extra - Ferramenta de Depuração Alternativa: `lldb`

Existem várias ferramentas de depuração para além do `gcc`, dos quais destacamos o `lldb` (abreviatura de *low-level debugger*).
O `lldb` é funcionalmente idêntico ao `gdb` e mais fácil de obter em macOS (vem incluido com as ferramentas de desenvolvimento).

**É recomendada a utilização do `gdb` e não se dará apoio à utilização `lldb` no decurso da cadeira!**

O ponto forte do `lldb` reside não em mais funcionalidade, mas sim na forma como é implementado.
Seguindo a filosofia do projeto LLVM, é modular, e partilha vários componentes (*parser*, avaliador de expressões, etc.) com o compilador `clang` do mesmo projeto, beneficiando de todo o investimento neles feito.

Para além disto, foi desenhado com integração em mente, expondo toda a funcionalidade por uma *API* (interface de programação de aplicações), seja para ferramentas simples de análise de binários, ou para um *IDE* (ambiente integrado de desenvolvimento) como o XCode.

Na [documentação oficial do `lldb`](https://lldb.llvm.org/index.html), encontra-se uma [referência de correspondências entre comandos do `gdb` e `lldb`](https://lldb.llvm.org/use/map.html).
As formas abreviadas dos comandos usados neste guião são quase todas iguais ao equivalente no `gdb`, mas as formas longas têm maior variação, por exemplo, `break <nome função>` corresponde a `breakpoint set --name <nome função>`, mas `b` funciona da mesma forma que no `gdb`).

----

Contactos para sugestões/correções: [LEIC-Alameda](mailto:leic-so-alameda@disciplinas.tecnico.ulisboa.pt), [LEIC-Tagus](mailto:leic-so-tagus@disciplinas.tecnico.ulisboa.pt), [LETI](mailto:leti-so-tagus@disciplinas.tecnico.ulisboa.pt)
