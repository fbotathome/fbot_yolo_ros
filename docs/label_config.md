# Configuração de Labels no `detect_3d_node`

Este documento explica como o `detect_3d_node` usa configurações específicas por `label`, quais labels já possuem tratamento dedicado e como adicionar uma nova label quando você quiser ajustar o comportamento do filtro 3D.

## O que são as definições específicas por label

No `detect_3d_node`, algumas classes de objetos recebem limites próprios de extensão em profundidade. Isso existe porque objetos diferentes têm geometrias diferentes e, portanto, tolerâncias diferentes para o quanto a nuvem de profundidade pode “abrir” no eixo `Z`.

Hoje, a configuração específica por label é usada em dois pontos:

1. `max_object_depth_extent` para o filtro de foreground.
2. Validação física da `BoundingBox3D` final.

Em outras palavras:

- o filtro de foreground usa o limite por label para decidir até onde manter pontos em profundidade;
- a validação final rejeita caixas cuja extensão em `Z` seja grande demais ou pequena demais para aquela classe.

## Labels configuradas hoje

As labels com configuração dedicada atualmente são:

- `bottle`
- `cup`
- `can`
- `person`
- `chair`
- `default`

Os valores definidos no código são estes:

- `bottle`: mínimo `0.03 m`, máximo `0.15 m`
- `cup`: mínimo `0.03 m`, máximo `0.12 m`
- `can`: mínimo `0.03 m`, máximo `0.10 m`
- `person`: mínimo `0.10 m`, máximo `0.55 m`
- `chair`: mínimo `0.05 m`, máximo `0.60 m`
- `default`: mínimo `0.01 m`, máximo `0.60 m`

## Onde isso fica configurado

As definições ficam em `detect_3d_node.py`:

- os parâmetros ROS são declarados no `__init__`
- os limites são carregados em `on_configure`
- o uso por label acontece em `_get_max_depth_extent()` e `_validate_bbox3d_extent()`

No launch, os parâmetros expostos hoje são:

- `max_depth_extent_default`
- `max_depth_extent_bottle`
- `max_depth_extent_cup`
- `max_depth_extent_can`
- `max_depth_extent_person`
- `max_depth_extent_chair`

Esses parâmetros podem ser passados em `yolo_bringup/launch/yolo.launch.py`.

## Como uma label é usada no código

Quando chega uma detecção 2D, o código pega `detection.label` e faz duas coisas:

1. converte a label para `lower()`
2. procura essa chave no dicionário `self.depth_extent_limits`

Se a label existir no dicionário, ela usa os limites específicos.
Se não existir, cai automaticamente em `default`.

Isso significa que a label precisa bater com a string esperada pelo pipeline, por exemplo:

- `Bottle` vira `bottle`
- `BOTTLE` vira `bottle`
- `bottle` continua `bottle`

## Como configurar uma nova label

Se você quiser adicionar uma nova classe, por exemplo `box`, siga este fluxo:

1. Adicione os parâmetros no `__init__` de `detect_3d_node.py`.
2. Adicione a entrada correspondente em `self.depth_extent_limits` dentro de `on_configure`.
3. Se quiser configurar via launch, adicione também um `LaunchConfiguration` e um `DeclareLaunchArgument` em `yolo.launch.py`.
4. Passe o novo parâmetro para o `Node` de `detect_3d_node`.
5. Ajuste os valores mínimo e máximo de acordo com a geometria real do objeto.

## Exemplo prático

Suponha que você queira adicionar a label `box` com faixa de profundidade esperada entre `0.04 m` e `0.25 m`.

### 1. Adicione o parâmetro no nó

No `__init__`:

```python
self.declare_parameter("max_depth_extent_box", 0.25)
```

### 2. Adicione no dicionário de limites

No `on_configure`:

```python
"box": (
    0.04,
    self.get_parameter("max_depth_extent_box").get_parameter_value().double_value,
),
```

### 3. Exponha no launch

Em `yolo_bringup/launch/yolo.launch.py`:

```python
max_depth_extent_box = LaunchConfiguration("max_depth_extent_box")
max_depth_extent_box_cmd = DeclareLaunchArgument(
    "max_depth_extent_box",
    default_value="0.25",
    description="Maximum expected depth extent for boxes",
)
```

E no `Node`:

```python
"max_depth_extent_box": max_depth_extent_box,
```

E no retorno da função:

```python
max_depth_extent_box_cmd,
```

## Boas práticas para escolher os valores

- `min` muito alto pode rejeitar objetos pequenos ou parcialmente vistos.
- `max` muito baixo pode cortar objetos legítimos e gerar falsos negativos.
- `max` muito alto deixa o filtro permissivo demais e reduz a proteção contra bleeding.
- Para objetos transparentes, como garrafas, o `max` costuma precisar ser mais restrito.
- Para objetos altos, como pessoas ou cadeiras, o `max` precisa ser maior.

## Resumo

As labels específicas servem para adaptar o filtro 3D ao formato real do objeto. Se a classe não tiver regra própria, o sistema usa `default`. Para adicionar uma nova label, você precisa declarar o parâmetro, registrar a faixa no dicionário de limites e, se quiser controle via launch, expor o mesmo parâmetro no `yolo.launch.py`.
