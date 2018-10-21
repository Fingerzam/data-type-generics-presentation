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

- Sum
  ```haskell
  data Sum a b = Inl a | Inr b
  ```
- Constructors
  ```haskell
  newtype Constructor (name :: Symbol) a = Constructor a
  ```
- No Arguments
  ```haskell
  data NoArguments = NoArguments
  ```
- Argument
  ```haskell
  newtype Argument a = Argument a
  ```
- Products
  ```haskell
  data Product a b = Product a b
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
- `NoConstructors`

# How to Write Such a Function?

::: incremental

- `Constructor`, `Sum`, `Product`, `Argument`, `NoArguments` and `NoConstructors`
  are all different types
- How to write a function that possibly works on all of them?
- Typeclasses

:::

# Eq

```haskell
class GenericEq a where
  genericEq' :: a -> a -> Boolean
```
. . .
```haskell
instance genericEqNoArguments
  :: GenericEq NoArguments where
  genericEq' NoArguments NoArguments = true
```
. . .
```haskell
instance genericEqConstructor
  :: GenericEq args
  => GenericEq (Constructor name args) where
  genericEq' (Constructor x) (Constructor y)
    = genericEq' x y
```
. . .
```haskell
> genericEq' (from Simple) (from Simple)
true
```

# Cannot Handle Sums yet

```haskell
> genericEq' (from Negative) (from Negative)
  No type class instance was found for
    Main.GenericEq (Sum (Constructor "Positive" NoArguments) 
                        (Constructor "Negative" NoArguments))
```

# Simple Sums

```haskell
instance genericEqSum
  :: (GenericEq a, GenericEq b)
  => GenericEq (Sum a b) where
  genericEq' (Inl x) (Inl y) = genericEq' x y
  genericEq' (Inr x) (Inr y) = genericEq' x y
  genericEq' _       _       = false
```
. . .
```haskell
> genericEq' (from Positive) (from Positive)
true
```
. . .
```haskell
> genericEq' (from Positive) (from Negative)
false
```

# Argument

```haskell
> genericEq' (from (UserName "KEKKONEN_3000")) 
             (from (UserName "KEKKONEN_3000"))
  No type class instance was found for
                                    
    Main.GenericEq (Argument String)
```

# Argument

```haskell
instance genericEqArgument
  :: Eq arg
  => GenericEq (Argument arg) where
  genericEq' (Argument x) (Argument y) = x == y
```
. . .
```haskell
> genericEq' (from (UserName "KEKKONEN_3000")) 
             (from (UserName "KEKKONEN_3000"))
true
```
. . .
```haskell
> genericEq' (from (UserName "Alice")) 
             (from (UserName "Bob"))
false
```

# Product of Arguments

```haskell
> genericEq' (from (Example2 "Alice" 1)) 
             (from (Example2 "Bob" 1))
  No type class instance was found for
    Main.GenericEq (Product (Argument String) 
                            (Argument String))
```

# Product of Arguments

```haskell
instance genericEqProduct
  :: (GenericEq a, GenericEq b)
  => GenericEq (Product a b) where
```
. . .
```haskell
  genericEq' (Product x1 y1) (Product x2 y2)
    = genericEq' x1 x2 && genericEq' y1 y2
```
. . .
```haskell
> genericEq' (from (Example2 "Alice" 1)) 
             (from (Example2 "Bob" 1))
false
```
. . .
```haskell
> genericEq' (from (Example2 "Alice" 1)) 
              (from (Example2 "Alice" 1))
true
```

# Stare in to the Void

- Should also cover empty types

```haskell
instance genericEqNoConstructors
  :: GenericEq NoConstructors where
  genericEq' _ _ = true
```

# Helper function

- To actually use this, write a helper function to do the generic conversion

```haskell
genericEq :: forall a rep
           . Generic a rep 
          => GenericEq rep 
          => a -> a -> Boolean
genericEq x y = genericEq' (from x) (from y)
```

# User code

- Now with all of this done, all that a user needs to do is write an instance that calls `genericEq`

```haskell
instance eqExample :: Eq Example where
  eq = genericEq
```
. . .
```haskell
> (Example2 "KEKKONEN_3000" 10) == (Example2 "KEKKONEN_3000" 10)
true
```
. . .
```haskell
> (Example2 "KEKKONEN_3000" 10) == (Example1 "KEKKONEN_3000" "10")
false

```

# Show must go on

```haskell
class GenericShow a where
  genericShow' :: a -> String
```
. . .
```haskell
instance genericShowConstructorNoArguments
  :: IsSymbol name
  => GenericShow (Constructor name NoArguments) where
```
. . .
```haskell
  genericShow' (Constructor NoArguments)
    = let constructorNameProxy = SProxy :: SProxy name
          constructorName = reflectSymbol constructorNameProxy
      in constructorName
```
. . .
```haskell
> log $ genericShow' $ from Simple
Simple
```

# Sums

```haskell
> log $ genericShow' $ from Positive
  No type class instance was found for
    Main.GenericShow (Sum (Constructor "Positive" NoArguments) 
                          (Constructor "Negative" NoArguments))
```
. . .
```haskell
instance genericShowSum
  :: (GenericShow a, GenericShow b)
  => GenericShow (Sum a b) where
```
. . .
```haskell
  genericShow' (Inl x) = genericShow' x
  genericShow' (Inr y) = genericShow' y
```
. . .
```haskell
> log $ genericShow' $ from Positive
Positive
```

# Single Argument

```haskell
> log $ genericShow' $ from (Just "Tampere")
  No type class instance was found for
    Main.GenericShow (Constructor "Just" (Argument String))
```
. . .
```haskell
instance genericShowConstructorArgument
  :: (Show arg, IsSymbol name)
  => GenericShow (Constructor name (Argument arg)) where
```
. . .
```haskell
  genericShow' (Constructor (Argument arg))
    = let constructorName = reflectSymbol (SProxy :: SProxy name)
          argStr = show arg
      in "(" <> constructorName <> " " <> argStr <> ")"
```
. . .
```haskell
> log $ genericShow' $ from (Just "Tampere")
(Just "Tampere")
```

# Multiple Arguments

```haskell
instance genericShowConstructorProduct
  :: (IsSymbol name, ??? a, ??? b)
  => GenericShow (Constructor name (Product a b))
```

# Helper Function to the Rescue

```haskell
class GenericShowArgs a where
  genericShowArgs :: a -> Array String
```
. . .

- Turn a product of arguments to an array of string

. . .
```haskell
instance genericShowArgsArgument
  :: Show a
  => GenericShowArgs (Argument a) where
```
. . .
```haskell
  genericShowArgs (Argument x) = [show x]
```
. . .
```haskell
instance genericShowArgsProduct
  :: (GenericShowArgs a, GenericShowArgs b)
  => GenericShowArgs (Product a b) where
```
. . .
```haskell
  genericShowArgs (Product x y)
    = genericShowArgs x <> genericShowArgs y
```

# Multiple Arguments

```haskell
instance genericShowConstructorProduct
  :: (IsSymbol name, GenericShowArgs (Product a b))
  => GenericShow (Constructor name (Product a b)) where
```
. . .
```haskell
  genericShow' (Constructor prod)
    = let constructorName = reflectSymbol (SProxy :: SProxy name)
          args = genericShowArgs prod
          argStr = intercalate " " args
      in "(" <> constructorName <> " " <> argStr <> ")"
```
. . .
```haskell
> log $ genericShow' $ from (Example2 "hep" 2)
(Example2 "hep" 2)
```

# Simplified

```haskell
instance genericShowArgsNoArguments
  :: GenericShowArgs NoArguments where
  genericShowArgs NoArguments = []
```
. . .
```haskell
instance genericShowConstructor
  :: (GenericShowArgs params, IsSymbol name)
  => GenericShow (Constructor name params) where
```
. . .
```haskell
  genericShow' (Constructor args) =
    let constructorNameProxy = SProxy :: SProxy name
        constructorName = reflectSymbol constructorNameProxy
        paramsStringArray = genericShowArgs args
    in case paramsStringArray of
      []       -> constructorName
      someArgs ->
        "(" <> constructorName <> " " 
            <> (intercalate " " someArgs) 
            <> ")"
```

# Helpers

```haskell
genericShow :: forall a rep
             . Generic a rep 
            => GenericShow rep 
            => a 
            -> String
genericShow x = genericShow' (from x)
```
. . .
```haskell
instance showExample :: Show Example where
  show = genericShow
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
- `NoContructors`

