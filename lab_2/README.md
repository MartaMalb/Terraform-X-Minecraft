# I am going to build a wall...

## Introducing locals and functions
By now, we should know how to create resources and use variables. In Terraform, we can also use locals which you cannot override from the CLI. There are different approaches and use cases for when you should use one or another, which we will not fully cover today, but one of the examples below will be a great use case for using locals. 

We will also leverage the Terraform functions to make our code cleaner and more reusable - placing hundreds of blocks, in the same manner as we did in the previous exercise, would be a tedious task. Therefore, we are going to use a `for_each` meta-argument and `for` expression in our first step - to learn more about what is available for you, see the [documentation](https://developer.hashicorp.com/terraform/language). Please go to your `main.tf` file and replace its content with the snippet below: 

```go
locals {
    list_of_blocks = [
        {
            x = 0
            y = -60
            z = 0
        },
        {
            x = 0
            y = -60
            z = 1
        },
        {
            x = 0
            y = -60
            z = 2
        },
        {
            x = 0
            y = -60
            z = 3
        },
        {
            x = 0
            y = -60
            z = 4
        }
    ]
}


resource "minecraft_block" "multiple_blocks" {
  for_each = { for i, o in local.list_of_blocks : "block-${i}" => o }

  material = var.block_material

  position = {
    x = each.value.x,
    y = each.value.y,
    z = each.value.z
  }
}
```

We've created a list of coordinates in our locals, and we use `for_each` to create a resource for every coordinate available. You can now run the command below followed by `yes`:

```bash
terraform apply
```
Terraform should attempt to remove your old line and make a new one on a different axis. Your output should look like below:

<p align="center">
  <img src="./images/for-each.png" />
</p>

You may need to refresh your map by running:

```bash
render-flat
```

## Fair square
In our next step, we will use `setproduct` and `range` functions to create the list of coordinates for us, rather than having to create it ourselves. Let's go to our `main.tf` again and modify the locals block.

```go
locals {
    list_of_coordinates = setproduct(range(var.width), [-60], range(var.length))
}
```
<b>Note</b>: We left -60 intentionally as this will define our ground level. 

Despite being provided with only one argument (integer), the Range function will produce a list of numbers within that range starting from 0 and not including the number. There is more to it, so if you feel adventurous see the [documentation](https://developer.hashicorp.com/terraform/language/functions/range).

The `setproduct` function will take multiple lists and create objects with all possible variations for the data in the lists - for more details refer to the [documentation](https://developer.hashicorp.com/terraform/language/functions/setproduct).

The above will give us list of coordinates looking like:

```go
[
    0,
    -60,
    0,
]
```
We now have all the values, but we are still missing the keys. We are going to use a `zipmap` function to solve this problem. More about `zipmap` in the [documentation](https://developer.hashicorp.com/terraform/language/functions/zipmap).

Add another line to your `locals` block in `main.tf` file so it looks like below:
```go
locals {
    list_of_coordinates = setproduct(range(var.width), [-60], range(var.length))
    list_of_blocks = [for block in local.list_of_coordinates : zipmap(["x", "y", "z"], block)]
}
```
In the above example, we use for loop to transform our lists of coordinates into the maps we can use.


Let's now move to our `variables.tf` file and define the `width` and the `length` of our line.

```go
variable "width" {
    type = number
    default = 10
}

variable "length" {
    type = number
    default = 1
}
```

In this case, we transform the data within the locals block and use it as the input. Anyone running/applying this configuration, whether from CLI or CI/CD pipeline, does not need to worry about how it is done, but rather focus on providing desired `width` and `length`.

Once your files are updated, you can run the `terraform apply` followed by `yes`.

Your output should look like this:

<p align="center">
  <img src="./images/functions.png" />
</p>

You may need to refresh your map by running:

```bash
render-flat
```
Now let's destroy our current creations by running:

```bash
terraform destroy
```
Followed by `yes`

## Terraform modules - let's get cubical
Now you could build a flat rectangle or a line by simply adjusting your width and length - this sounds like something you could reuse in the future, right? That is a great use case for using Terraform modules. This should be fairly quick, as most of the inputs are already defined by the variables with the default values.
First, let's create a new directory and mv our current code by executing the following commands in your terminal:

```bash
mkdir /home/playground/workdir/Terraform-X-Minecraft/cubical
cp terraform.tf cubical/
mv main.tf variables.tf cubical/
```
Now, in your IDE create new empty main.tf and variables.tf files, and copy the following to the `main.tf`:
```go
module "line" {
    source = "./cubical"
}
```
Now you can run `terraform init` and `terraform plan` (make sure you are in `/home/playground/workdir/Terraform-X-Minecraft` directory), and you should see the same output as before. Before applying anything, we should improve our module making it even more well-rounded. Let's look at our `cubical/main.tf` file first, and update the `locals` block to look like the one below:

```
locals {
    list_of_coordinates = setproduct(range(var.width_start, var.width_stop, var.width_step), [var.height], range(var.length_start, var.length_stop, var.length_step))
    list_of_blocks = [for block in local.list_of_coordinates : zipmap(["x", "y", "z"], block)]
}
```
We added extra variables to take full control of the size with start, stop and step variables for width and length. Don't forget we can also build on different heights. Now it's time to define them in `cubical/variables.tf`. Your file should look like this:
```go
variable "block_material" {
    type = string
    default = "minecraft:stone"
    description = "Type of material you are using for your structure - different materials will have different colours"
}

variable "width_start" {
    default = 0 
    description = "starting X coordinate of the range"
}

variable "width_stop" {
    default = 1 
    description = "last X coordinate of the range"

}

variable "width_step" {
    default = 1 
    description = "number of steps to find the next X value in the range i.e. with step 1 range 0 to 3 will include 0,1,2 with step 2 it will include 0, 2"
}

variable "length_start" {
    default = 0 
    description = "starting Z coordinate of the range"

}

variable "length_stop" {
    default = 1 
    description = "last Z coordinate of the range"
}

variable "length_step" {
    default = 1 
    description = "number of steps to find the next Z value in the range i.e. with step 1 range 0 to 3 will include 0,1,2 with step 2 it will include 0, 2"

}

variable "height" {
    default = -60 
    description = "Value of the Y coordinate which is height - on the flat map we provided -60 is the ground level"
}

```
Now run `terraform init` and `terraform plan` again. Your output shows that your module is trying by default to build one block starting from coordinates 0,-60,0. We can now move to our `main.tf` file in the top directory and change the way we call our module, you can use the snippet below:

```go
module "square" {
    source = "./cubical"
    length_stop = 5
    width_stop = 5
}
```
That should give us a square built on the ground level, starting from (0,-60,0) to (5,-60,5) coordinates - let's run:
```bash
terraform init
terraform apply
```
Followed by `yes` and then, shortly after, we can refresh the map to see the results by running:
```
reneder-flat
```

## Modules and functions - count
We all know where this is going - we have a module to create flat squares, and we can stack them on top of each other: time to make a cube! This time we are going to count another Terraform meta argument that is yet to be covered today. We are going to call our module 5 times, and change the height coordinate each time we are calling the module:

```go
module "cube" {
    count = 5
    source = "./cubical"
    length_stop = 5
    width_stop = 5
    height = -60 + count.index
}
```
Now let's run `terraform init` and  `terraform apply`, followed by `yes`. At this point, refresh the map and admire the results!

This was lab number 2! You should know your way around Terraform by now - in the next lab we will provide you with some additional tools, and you will take it wherever you want! Go to [Lab_3 -  Time to get creative](../lab_3/README.md) when you are ready!
