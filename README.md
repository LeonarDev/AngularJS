# Angular 1.8.x
https://angularjs.org/

Este repositório tem o objetivo de abordar as principais `práticas para otimizar a performance` do AngularJS `visando a sustentação de sistemas legados`.


<hr>
<br>


## O problema
O AngularJS utiliza uma técnica chamada de `Dirty Checking` como fundamento para o seu funcionamento. Esta técnica envolve um laço constante, chamado de `ciclo digestivo`, que verifica o tempo inteiro se houve alguma mudança no escopo da aplicação, ou seja, o AngularJS está o tempo inteiro perguntando se houve alguma mudança nos valores de variáveis e funções. Se houver mudança, ele atualiza todo o escopo com novos valores.

Para atingir este objetivo, o AngularJS lança mão de um recurso chamado de `Watcher` que é encarregado de monitorar todas as alterações ocorridas no escopo. Eles são criados aos montes para diretivas diversas como `ng-show`, `ng-if`, `ng-repeat`, `ng-model` e tantas outras. Quanto mais watchers sendo usados pela aplicação mais o ciclo digestivo demora para ser completado, e quanto mais ele demora, mais lento fica a aplicação.

As técnicas de otimização visam reduzir o número de watchers criados na aplicação a fim de conservar o ciclo digestivo o mais rápido possível.


<hr>
<br>


## Técnica de Bind Once
A técnica de `bind once` tem o objetivo de desabitar o recurso de `two way data biding` nativo do AngularJS. O `two way data binding` é responsável por monitorar as mudanças em uma variável e refleti-la na tela imediatamente. Por exemplo:

```js
<div class="row">
    <div class="col-md-12">
        <h2> {{ vm.titulo }} </h2>
    </div>
</div>
```

Neste exemplo, estamos imprimindo o valor da variável `titulo` na tag `h2` do HTML. Caso a variável receba um novo valor seja no controlador ou por alguma interação do usuário, o novo valor será refletido automaticamente. Isto acontece, porque quando o Angular imprime o valor da variável `titulo`, ele mantém um `watcher` amarrado a variável verificando se ela sofrerá alguma alteração.

Na maioria das vezes não temos a necessidade de manter este `watcher`. As variáveis uma vez impressas na tela não precisam de alterações dinâmicas. Com isso podemos fazer uso da técnica de `bind once` que serve justamente para remover este `watcher` e para isso basta adicionar `::` antes da variável a ser impressa. Por exemplo:

```js
<div class="row">
    <div class="col-md-12">
        <h2> {{ ::vm.titulo }} </h2>
    </div>
</div>
 
<tr ng-repeat="registro in ::vm.registros track by registro.Codigo">
```

Com isso deixaremos nossa aplicação mais leve e mais rápida.

<br>

> ### Exceções
> Existem algumas exceções para esta regra. O bind once NÃO precisa ser utilizado nos seguintes casos:
> 
> - Em grids com edição inline.
> - Em grids com atualização de dados.
>     - Apenas os campos que precisarem ser atualizados deverão ser Two-way data biding.
>     - Se todos os campos sofrerem atualização, então nenhum deles precisa ser bind once.
> - Em qualquer interface que sofra atualização por parte de um ação do usuário.


<hr>
<br>


## Técnica de track by no ng-repeat
Imagine que temos um vetor de registros para imprimir numa lista. Como no exemplo abaixo:

```js
<tr ng-repeat="registro in ::vm.registros">
    <td>{{ ::registro.Codigo }}</td>
    <td>{{ ::registro.Descricao }}</td>
</tr>
```

Para cada tag `tr` que será replicada pelo loop, o Angular irá criar uma variável interna chamada `$$hashKey` que servirá para identificar unicamente o elemento DOM.
O problema é que se nosso vetor de registros sofrer alguma alteração, adição ou remoção, o valor de `$$hashKey` será atualizado e consequentemente todos os elementos do DOM também irão.

Na prática, sem um identificador único para os elementos gerados, o custo das operações no DOM será muito alto, porque a cada alteração todos eles deverão ser gerados outra vez.

Para otimizar estes casos, podemos utilizar o recurso chamado de `track by`. Observe o mesmo exemplo:

```js
<tr ng-repeat="registro in ::vm.registros track by registro.Codigo">
    <td>{{ ::registro.Codigo }}</td>
    <td>{{ ::registro.Descricao }}</td>
</tr>
```

Com o `track by` nós retiramos a responsabilidade do AngularJS de criar os identificadores únicos e oferecemos o nosso próprio identificador. Agora quando nosso vetor de registros sofrer, por exemplo, uma adição, em vez de reconstruir todos os elementos do DOM, o Angular só fará operações sobre o novo registro.
Caso o vetor de registros não tenha um atributo único como um ID ou Código, podemos usar a variável interna `$index` do `ng-repeat`. Assim:

```js
<tr ng-repeat="registro in ::vm.registros track by $index">
    <td>{{ ::registro.Codigo }}</td>
    <td>{{ ::registro.Descricao }}</td>
</tr>
```


<hr>
<br>


## $watchCollection no lugar de $watch para coleções
O AngularJS nos oferece um forma de criar nossos próprios `watchers` para quando temos a necessidade de capturar as mudanças realizadas em uma variável ou coleção de objetos. Esta forma é a função $watch. Por exemplo:

```js
$scope.$watch(`vm.registro`, function(novoValor, velhoValor) {
    //Fazer alguma coisa aqui
});
 
$scope.$watch(function() {
    return vm.registro;
}, function(novoValor, velhoValor) {
    //Fazer alguma coisa aqui
});
```

A grosso modo, quando precisamos observar mudanças em toda árvore do objeto dentro de uma coleção, podemos utilizar o terceiro parâmetro do $watch passando o valor `true`:

```js
$scope.$watch(`vm.registro`, function(novoValor, velhoValor) {
    //Fazer alguma coisa aqui
}, true);
 
$scope.$watch(function() {
    return vm.registro;
}, function(novoValor, velhoValor) {
    //Fazer alguma coisa aqui
}, true);
```

Com isso ela fará uma varredura em profundidade que pode ser uma operação, dependendo do tamanho da coleção, bastante custosa.  Nestes casos, podemos utilizar a `$watchCollection`. Como no exemplo:

```js
$scope.$watchCollection(`vm.registros`, function(novoValor, velhoValor) {
    //Fazer alguma coisa aqui
});
 
$scope.$watchCollection(function() {
    return vm.registros;
}, function(novoValor, velhoValor) {
    //Fazer alguma coisa aqui
});
```

A função `$watchColletion` é otimizada para estes casos e seu custo é bem menor do que utilizar `$watch` com o terceiro parâmetro como `true`.


<hr>
<br>


## ng-if ao invés de ng-show/ng-hide
A principal diferença entre o `ng-if` e `ng-show/ng-hide` está no fato de o primeiro remover o elemento do DOM enquanto o segundo apenas aplica uma classe CSS para escondê-lo. Para a grande maioria dos casos, o `ng-if` pode assumir o papel e desempenhar o mesmo comportamento com a vantagem de deixar o DOM mais limpo.

Exemplo:

```js
<div class="row` ng-show="vm.carregando">
    <h4>Carregando tela...</h4>
</div>
 
<!-- Pode ser substituído por -->
 
<div class="row` ng-if="vm.carregando">
    <h4>Carregando tela...</h4>
</div>
```

Quando a variável `vm.carregando` receber o valor `false`, o elemento `div` com a classe `row` será removido do DOM.
Talvez para computadores com um processamento razoável a vantagem de usar esta abordagem não seja tão perceptível, porém quando pensamos em dispositivos móveis, seja em um aplicativo híbrido ou um web app, manter o DOM enxuto pode aumentar a performance geral da aplicação.


<hr>
<br>


## $scope.$digest() no lugar de $scope.$apply()
Como o Angular trabalha com um ciclo digestivo próprio composto por uma variedade de observadores que atualizam informações o tempo todo, quando formos utilizar bibliotecas terceiras dentro de nossa aplicação, precisamos encontrar uma forma de notificar o Angular sobre as alterações realizadas por elas. Uma dessas formas é utilizar a função `$scope.apply()` ou a `$scope.digest()`.

Na prática, se estivermos usando algum componente em jQuery que altera informações do escopo do angular, precisamos invocar um destas duas funções para iniciar o ciclo digestivo programaticamente. Hoje a maioria dos componentes em jQuery possuem uma versão na forma de diretiva para o AngularJS.

Por exemplo:

```js
// No controlador:

vm.registro.nome = `TESTE`;

var elemento = $(`#input-nome`);

elemento.val(`TESTE NOVO`);

```

```js
// No HTML:

<input id="input-nome` type="text` ng-model="vm.registro.nome">
```

Neste caso, temos um seletor jQuery fazendo uma alteração direta no valor do input, porém nossa variável `vm.registro.nome`, que é o `ng-model` do input, não recebe o novo valor. Este é um típico caso onde existe a necessidade de iniciar o ciclo digestivo do AngularJS programaticamente.

A diferença entre o $scope.$apply() e o $scope.digest() é que o primeiro inicia o ciclo a partir do $rootScope e varre todos os escopos da aplicação buscando por alterações. Já o segundo, parte do escopo atual, na maioria dos casos, do escopo do controlador ou de seus subsequentes escopos filhos. Ou seja, a função $scope.$digest() é mais rápida, porque tem menos escopos para varrer.

Atualmente, a maioria destes casos são facilmente resolvidos com a criação de diretivas para encapsular bibliotecas terceiras, no entanto, caso surja a necessidade de acionar o ciclo digestivo do AngularJS, a opção mais rápida é através de `$scope.$digest()`.

<hr>
<br>


## Conclusão
Vimos que algumas técnicas simples fazem muita diferença na performance da aplicação final. Seguindo estas recomendações, manteremos nossas aplicações funcionando de forma eficiente.
