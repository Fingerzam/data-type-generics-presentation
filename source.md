---
title: Data Type Generics
subtitle: Type Safety Without Boilerplate
author: Juhana Laurinharju
institute: Emblica Oy
date: \today
theme: Madrid
colortheme: dolphin
fontfamily: noto-sans
header-includes:
- \usepackage{cmbright}
fontsize: 10pt
---

# Algebraic Data Types are Cool

```haskell
data UserType    = Regular | Admin

newtype UserName = UserName String

data User        = User UserType UserName
```

# Type classes give polymorphism

```haskell
instance showUserType :: Show UserType where
  show Regular = "Regular"
  show Admin   = "Admin"
```
. . .
```haskell
instance showUserName :: Show UserName where
  show (UserName name) = "(UserName " <> name <> ")"
```
. . .
```haskell
instance showUser :: Show User where
  show (User userType userName) =
    "(User " <> show userType <> " " <> show userName <> ")"
```

# But that's a lot of work

- Could we somehow look at the structure of our types?

# Anatomy of Algebraic Data Types

```haskell
data Example = Example1 String String
             | Example2 String Int
```

. . .

::: incremental

- Sums
- Contructors
- Products

:::

# Encoded as Types

. . .

::: incremental

- Constructors
  ```haskell
  newtype Constructor (name :: Symbol) a = Constructor a
  ```
- Products
  ```haskell
  data Product a b = Product a b
  ```
- Sum
  ```haskell
  data Sum a b = Inl a | Inr b
  ```
- Argument
  ```haskell
  newtype Argument a = Argument a
  ```
- No Arguments
  ```haskell
  data NoArguments = NoArguments
  ```
- Empty types
  ```haskell
  data NoConstructors
  ```

:::

# To There and Back

```haskell
class Generic a rep | a -> rep where
  to   :: rep -> a
  from :: a   -> rep
```

. . .

- Compiler can generate you instances

  . . .
  ```haskell
  derive instance genericMyType :: Generic MyType _
  ```
  
. . .

- But we should check out some examples

# Extremely Simple Type

```haskell
data Simple = Simple
```
. . .
```haskell
instance genericSimple
  :: Generic Simple (Constructor "Simple" NoArguments) where
```
. . .
```haskell
  from Simple = Constructor NoArguments
```
. . .
```haskell
  to (Constructor NoArguments) = Simple
```

# Simple Sum Type

```haskell
data Vote = Positive | Negative
```
. . .
```haskell
instance genericVote
  :: Generic Vote (Sum (Constructor "Positive" NoArguments)
                       (Constructor "Negative" NoArguments)) where
```
. . .
```haskell
  from Positive = Inl (Constructor NoArguments)
  from Negative = Inr (Constructor NoArguments)
```
. . .
```haskell
  to (Inl (Constructor NoArguments)) = Positive
  to (Inr (Constructor NoArguments)) = Negative
```

# Simple Product Type

```haskell
data Vec3 = Vec3 Number Number Number
```
. . .
```haskell
instance genericVec3
  :: Generic Vec3 (Constructor "Vec3"
                    (Product
                      (Argument Number)
                      (Product
                        (Argument Number)
                        (Argument Number)))) where
```
. . .
```haskell
  from (Vec3 a b c)
    = Constructor
       (Product (Argument a)
                (Product (Argument b)
                         (Argument c)))
```
. . .
```haskell
  to (Constructor
       (Product (Argument a)
                (Product (Argument b)
                         (Argument c)))) = Vec3 a b c
```

# Sum of Products

```haskell
data Example = Example1 String String
             | Example2 String Int
```
. . .
```haskell
instance genericExample
  :: Generic Example (Sum (Constructor "Example1"
                            (Product (Argument String)
                                     (Argument String)))
                          (Constructor "Example2"
                            (Product (Argument String)
                                     (Argument Int)))) where
```
. . .
```haskell
  from (Example1 str1 str2) 
    = (Inl (Constructor (Product (Argument str1) (Argument str2))))
  from (Example2 str int)   
    = (Inr (Constructor (Product (Argument str)  (Argument int))))
```
. . .
```haskell
  to (Inl (Constructor (Product (Argument str1) (Argument str2)))) 
    = Example1 str1 str2
  to (Inr (Constructor (Product (Argument str)  (Argument int))))  
    = Example2 str int
```

# Using generic functions

- There are functions that work with these generic representations

  . . .
  ```haskell
  > log $ genericShow' $ from $ Example1 "Hello" "Tampere"
  ```
  . . .
  ```haskell
  (Example1 "Hello" "Tampere")
  ```
  . . .
  ```haskell
  > log $ genericShow' $ from $ Just "Tampere"
  ```
  . . .
  ```haskell
  (Just "Tampere")
  ```
. . .
  
- And they are type checked

  . . .
  ```haskell
  newtype NoShow a = NoShow a
  ```
  . . .
  ```haskell
  > log $ genericShow' $ from $ Just (NoShow "Tampere")
  ```
  . . .
  ```haskell
    No type class instance was found for
                                    
      Data.Show.Show (NoShow String)
  ```

# Shape of ADTs

```haskell
Sum      (Constructor "Name1"          NoArguments)
    (Sum (Constructor "Name2"          (Argument Int))
         (Constructor "Name3" (Product (Argument String)
                                       (Argument Number))))
```

. . .

- `Sum` of
  - `Constructor` of
    - `NoArguments`
    - a single `Argument`
    - `Product` of `Argument`s

# How to Write Such a Function?

::: incremental

- `Constructor`, `Sum`, `Product`, `Argument`, `NoArguments` and `NoConstructors`
  are all different types
- How to write a function that possibly works on all of them?
- Typeclasses

:::

# Eq

```haskell
class GenerikEq a where
  generikEq' :: a -> a -> Boolean
```
. . .
```haskell
instance generikEqConstructor
  :: GenerikEq args
  => GenerikEq (Constructor name args) where
  generikEq' (Constructor x) (Constructor y)
    = generikEq' x y
```
. . .
```haskell
instance generikEqNoArguments
  :: GenerikEq NoArguments where
  generikEq' NoArguments NoArguments = true
```
. . .
```haskell
> generikEq' (from Simple) (from Simple)
true
```

# Cannot Handle Sums yet

```haskell
> generikEq' (from Negative) (from Negative)
  No type class instance was found for
    Main.GenerikEq (Sum (Constructor "Positive" NoArguments) 
                        (Constructor "Negative" NoArguments))
```

# Simple Sums

```haskell
instance generikEqSum
  :: (GenerikEq a, GenerikEq b)
  => GenerikEq (Sum a b) where
  generikEq' (Inl x) (Inl y) = generikEq' x y
  generikEq' (Inr x) (Inr y) = generikEq' x y
  generikEq' _       _       = false
```
. . .
```haskell
> generikEq' (from Positive) (from Positive)
true
```
. . .
```haskell
> generikEq' (from Positive) (from Negative)
false
```

# Argument

```haskell
> generikEq' (from (UserName "KEKKONEN_3000")) 
             (from (UserName "KEKKONEN_3000"))
  No type class instance was found for
                                    
    Main.GenerikEq (Argument String)
```

# Argument

```haskell
instance generikEqArgument
  :: Eq arg
  => GenerikEq (Argument arg) where
  generikEq' (Argument x) (Argument y) = x == y
```
. . .
```haskell
> generikEq' (from (UserName "KEKKONEN_3000")) 
             (from (UserName "KEKKONEN_3000"))
true
```
. . .
```haskell
> generikEq' (from (UserName "Alice")) 
             (from (UserName "Bob"))
false
```

# Product of Arguments

```haskell
> generikEq' (from (Example2 "Alice" 1)) 
             (from (Example2 "Bob" 1))
  No type class instance was found for
    Main.GenerikEq (Product (Argument String) 
                            (Argument String))
```

# Product of Arguments

```haskell
instance generikEqProduct
  :: (GenerikEq a, GenerikEq b)
  => GenerikEq (Product a b) where
```
. . .
```haskell
  generikEq' (Product x1 y1) (Product x2 y2)
    = generikEq' x1 x2 && generikEq' y1 y2
```
. . .
```haskell
> generikEq' (from (Example2 "Alice" 1)) 
             (from (Example2 "Bob" 1))
false
```
. . .
```haskell
> generikEq' (from (Example2 "Alice" 1)) 
              (from (Example2 "Alice" 1))
true
```

# Stare in to the Void

```haskell
instance generikEqNoConstructors
  :: GenerikEq NoConstructors where
  generikEq' _ _ = true
```

# Show must go on

```haskell
class GenerikShow a where
  generikShow' :: a -> String
```
. . .
```haskell
instance generikShowConstructorNoArguments
  :: IsSymbol name
  => GenerikShow (Constructor name NoArguments) where
```
. . .
```haskell
  generikShow' (Constructor NoArguments)
    = let constructorNameProxy = SProxy :: SProxy name
          constructorName = reflectSymbol constructorNameProxy
      in constructorName
```
. . .
```haskell
> log $ generikShow' $ from Simple
Simple
```

# Sums

```haskell
> log $ generikShow' $ from Positive
  No type class instance was found for
    Main.GenerikShow (Sum (Constructor "Positive" NoArguments) 
                          (Constructor "Negative" NoArguments))
```
. . .
```haskell
instance generikShowSum
  :: (GenerikShow a, GenerikShow b)
  => GenerikShow (Sum a b) where
```
. . .
```haskell
  generikShow' (Inl x) = generikShow' x
  generikShow' (Inr y) = generikShow' y
```
. . .
```haskell
> log $ generikShow' $ from Positive
Positive
```

# Single Argument

```haskell
> log $ generikShow' $ from (Just "Tampere")
  No type class instance was found for
    Main.GenerikShow (Constructor "Just" (Argument String))
```
. . .
```haskell
instance generikShowConstructorArgument
  :: (Show arg, IsSymbol name)
  => GenerikShow (Constructor name (Argument arg)) where
```
. . .
```haskell
  generikShow' (Constructor (Argument arg))
    = let constructorName = reflectSymbol (SProxy :: SProxy name)
          argStr = show arg
      in "(" <> constructorName <> " " <> argStr <> ")"
```
. . .
```haskell
> log $ generikShow' $ from (Just "Tampere")
(Just "Tampere")
```

# Multiple Arguments

```haskell
instance generikShowConstructorProduct
  :: (IsSymbol name, ??? a, ??? b)
  => GenerikShow (Constructor name (Product a b))
```

# Helper Function to the Rescue

```haskell
class GenerikShowArgs a where
  generikShowArgs :: a -> Array String
```
. . .

- Turn a product of arguments to an array of string

. . .
```haskell
instance generikShowArgsArgument
  :: Show a
  => GenerikShowArgs (Argument a) where
```
. . .
```haskell
  generikShowArgs (Argument x) = [show x]
```
. . .
```haskell
instance generikShowArgsProduct
  :: (GenerikShowArgs a, GenerikShowArgs b)
  => GenerikShowArgs (Product a b) where
```
. . .
```haskell
  generikShowArgs (Product x y)
    = generikShowArgs x <> generikShowArgs y
```

# Multiple Arguments

```haskell
instance generikShowConstructorProduct
  :: (IsSymbol name, GenerikShowArgs (Product a b))
  => GenerikShow (Constructor name (Product a b)) where
```
. . .
```haskell
  generikShow' (Constructor prod)
    = let constructorName = reflectSymbol (SProxy :: SProxy name)
          args = generikShowArgs prod
          argStr = intercalate " " args
      in "(" <> constructorName <> " " <> argStr <> ")"
```
. . .
```haskell
> log $ generikShow' $ from (Example2 "hep" 2)
(Example2 "hep" 2)
```

# Simplified

```haskell
instance generikShowArgsNoArguments
  :: GenerikShowArgs NoArguments where
  generikShowArgs NoArguments = []
```
. . .
```haskell
instance generikShowConstructor
  :: (GenerikShowArgs params, IsSymbol name)
  => GenerikShow (Constructor name params) where
```
. . .
```haskell
  generikShow' (Constructor args) =
    let constructorNameProxy = SProxy :: SProxy name
        constructorName = reflectSymbol constructorNameProxy
        paramsStringArray = generikShowArgs args
    in case paramsStringArray of
      []       -> constructorName
      someArgs ->
        "(" <> constructorName <> " " 
            <> (intercalate " " someArgs) 
            <> ")"
```

# Helpers

```haskell
generikShow :: forall a rep
             . Generic a rep 
            => GenerikShow rep 
            => a 
            -> String
generikShow x = generikShow' (from x)
```
. . .
```haskell
instance showExample :: Show Example where
  show = generikShow
```
. . .
```haskell
> log $ show $ (Example2 "hep" 2)
(Example2 "hep" 2)
```

# Recap

- `Sum` of
  - `Constructor` of
    - `NoArguments`
    - a single `Argument`
    - `Product` of `Argument`s

