---
title: "Save trees in a relational Database"
date: 2021-03-20T08:32:47+01:00
draft: false
---
# Store trees in a relational database efficiently
If you have some kind of tree in your application you should store it in a database specialized in graphs and trees, right? It depends. What if it is only a small tree and it is your only tree and you just don't want to manage a graph database just for this single graph which barely changes? Or what if you are just too lazy? Or what if you are not too lazy but you just don't care? Then I might have a solution for you.

![It's not that I'm lazy. It's that I just don't care meme](/img/dont_care.jpg)

## The problem
I wanted to write an application where the user can put things into categories. A category can have multiple sub categories. But since this would be my only use case for a tree and since categories don't change too often I wanted to stuff this into the same relational database like all the other data. Since the categories do not change very often we can calculate the tree once and cache it's output for some time.

There are multiple approaches, but most have the problem, that building the tree is super slow or need a lot of loops. So it would be great to have some sort of order where we need to loop over all the branches and leaves only once. That would be efficient enough for this usecase.

## The solution
Let's say we have a category with a name and each category can have one parent:
```python
class Category:
    name: str
    parent: Category = None

    children: list[Category] = None # for easier traversal
```
For the example let's assume the following tree:
```
└── recipes
    ├── dessert
    │   ├── jelly
    │   └── pudding
    │       ├── chocolat_pudding
    │       └── vanilla_pudding
    ├── main
    │   ├── curry
    │   │   └── mango_curry
    │   └── potatos
    └── starter
        ├── salad
        │   └── cesar_salad
        └── spring_roll
```
Categories could then look like this:
```python
recipes = Category(name='recipes')
dessert = Category(name='dessert', parent=recipes)
```
This way, we could save the tree in the correct order and just fetch it ordered by id. But that would break as soon as a new category gets added. One way would be to represent the categories position in the tree by a string and use it as attribute to sort by. 

The above mentioned category object would look like this:
```python
class Category:
    name: str
    tree_position: str
    parent: Category = None

recipes = Category(name='recipes', tree_position='A')
dessert = Category(name='dessert', tree_position='AA', parent=recipes)
jelly = Category(name='jelly', tree_position='AAA', parent=dessert)
pudding = Category(name='pudding', tree_position='AAB', parent=dessert)
chocolat_pudding = Category(name='chocolat pudding', tree_position='AABA', parent=pudding)
vanilla_pudding = Category(name='vanilla pudding', tree_position='AABB', parent=pudding)
main = Category(name='main', tree_position='AB', parent=recipes)
```
and so on and so forth.

The advantages are, that we can sort by the additional `tree_position` attribute and that we can quite easy see where in the tree a branch or leaf is situated. The disadvantage is, that moving a branch to another branch is quite expensive, since the childrens path needs to be recalculated. To keep it clean (like all the letters in subsequent order) the siblings need to be recalculated, too.

### Building the tree
With your data prepared like this you could query the data in order:
```SQL
SELECT * FROM categories ORDER BY tree_position;
```
Since shorter strings are considered "lower" than longer strings when otherwise equal all the children are right of the parents when sorted. A returned list of `tree_position`s could look like this:
```
A
AA
AAA
AAB
AB
ABA
ABB
```
So a branch is followed up to its leaves before the siblings get processed. With this information we could build a tree as follows:
```python
def add_children(category: Category, category_rows: list[CategoryRow]) -> None:
    if category_rows:
        current_row = category_rows[0]
        current_row_category = Category(name=current_row.name, tree_position=current_row.tree_position)
        current_tree_depth = len(current_row.tree_position)

        if current_tree_depth > len(category.tree_position):
            current_row_category.parent = category
            child_category = current_row_category
            category.children.append(child_category)
            add_children(child_category, category_rows[1:])
        elif current_tree_depth == len(category.tree_position):
            current_row_category.parent = category.parent
            sibling_category = current_row_category
            category.parent.children.append(sibling_category)
            add_children(sibling_category, category_rows[1:])
        else:
            current_row_category.parent = category.parent.parent
            parents_sibling_category = current_row_category
            category.parent.parent.children.append(parents_sibling_category)
            add_children(parents_sibling_category, category_rows[1:])

first_category_row: CategoryRow = sorted_category_rows[0]
root_category: Category = Category(name=first_category_row.name, tree_position=first_category_row.tree_position)
add_children(root_category, sorted_category_rows[1:])
```
This function visits every branch only once. With this function we would get a valid tree which looks like the following json representation:
```json
{
  "name": "recipes",
  "tree_position": "A",
  "children": [
    {
      "name": "dessert",
      "tree_position": "AA",
      "children": [
        {
          "name": "jelly",
          "tree_position": "AAA",
          "children": []
        },
        {
          "name": "pudding",
          "tree_position": "AAB",
          "children": [
            {
              "name": "chocolat_pudding",
              "tree_position": "AABA",
              "children": []
            },
            {
              "name": "vanilla_pudding",
              "tree_position": "AABB",
              "children": []
            }
          ]
        },
        {
          "name": "main",
          "tree_position": "AB",
          "children": [
            {
              "name": "curry",
              "tree_position": "ABA",
              "children": [
                {
                  "name": "mango_curry",
                  "tree_position": "ABAA",
                  "children": []
                }
              ]
            },
            {
              "name": "potatos",
              "tree_position": "ABB",
              "children": []
            }
          ]
        },
        {
          "name": "starter",
          "tree_position": "AC",
          "children": [
            {
              "name": "salad",
              "tree_position": "ACA",
              "children": [
                {
                  "name": "cesar_salad",
                  "tree_position": "ACAA",
                  "children": []
                }
              ]
            },
            {
              "name": "spring_roll",
              "tree_position": "ACB",
              "children": []
            }
          ]
        }
      ]
    }
  ]
}
```
This JSON output can now be stored in a cache to be delivered from there and does not have to be recalculated too often.

If you like you can [see the full code example here](https://gist.github.com/protux/7bddd2629f922641ba4acb0ad32c5848).
