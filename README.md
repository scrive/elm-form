# Elm Form

[![Elm CI](https://github.com/scrive/elm-form/workflows/Elm%20CI/badge.svg)](https://github.com/scrive/elm-form/actions)

> This is a fork from [etaque/elm-form](https://github.com/etaque/elm-form) compatible with `elm-explorations/elm-test@2.2.0`

HTML live form builders and validation for Elm.

    elm package install scrive/elm-form

For when the classical "a message per field" doesn't work well for you, at the price of losing some type safety (field names are made of strings, see [#97](https://github.com/etaque/elm-form/issues/97)).

## Features

- Validation API similar to `Json.Decode` with the standard `map`, `andThen`, etc: you either get the desired output value or all field errors
- HTML inputs helpers with pre-wired handlers for live validation
- Suite of basic validations, with a way to add your own
- Unlimited fields, see `andMap` function (as in `Json.Extra`)
- Nested fields (`foo.bar.baz`) and lists (`todos.1.checked`) enabling rich form build

## Basic usage

See the [example validation test suite](https://github.com/scrive/elm-form/blob/master/tests/ValidationTests.elm)
and [test helper function docs](http://package.elm-lang.org/packages/scrive/elm-form/latest/Form-Test)
for how to test-drive validations.

```elm
module Main exposing (Foo, Model, Msg(..), app, formView, init, update, validate, view)

import Browser
import Form exposing (Form)
import Form.Input as Input
import Form.Validate as Validate exposing (..)
import Html exposing (..)
import Html.Attributes exposing (..)
import Html.Events exposing (..)



-- your expected form output


type alias Foo =
    { bar : String
    , baz : Bool
    }



-- Add form to your model and msgs


type alias Model =
    { form : Form () Foo }


type Msg
    = NoOp
    | FormMsg Form.Msg



-- Setup form validation


init : Model
init =
    { form = Form.initial [] validate }


validate : Validation () Foo
validate =
    succeed Foo
        |> andMap (field "bar" email)
        |> andMap (field "baz" bool)



-- Forward form msgs to Form.update


update : Msg -> Model -> Model
update msg ({ form } as model) =
    case msg of
        NoOp ->
            model

        FormMsg formMsg ->
            { model | form = Form.update validate formMsg form }



-- Render form with Input helpers


view : Model -> Html Msg
view { form } =
    Html.map FormMsg (formView form)


formView : Form () Foo -> Html Form.Msg
formView form =
    let
        -- error presenter
        errorFor field =
            case field.liveError of
                Just error ->
                    -- replace toString with your own translations
                    div [ class "error" ] [ text (Debug.toString error) ]

                Nothing ->
                    text ""

        -- fields states
        bar =
            Form.getFieldAsString "bar" form

        baz =
            Form.getFieldAsBool "baz" form
    in
    div []
        [ label [] [ text "Bar" ]
        , Input.textInput bar []
        , errorFor bar
        , label []
            [ Input.checkboxInput baz []
            , text "Baz"
            ]
        , errorFor baz
        , button
            [ onClick Form.Submit ]
            [ text "Submit" ]
        ]


app =
    Browser.sandbox
        { init = init
        , update = update
        , view = view
        }

```

## Advanced usage

### Custom inputs

- For rendering, `Form.getFieldAsString`/`Bool` provides a `FieldState` record with all required fields (see package doc).

- For event handling, see all field related messages in `Form.Msg` type.

Overall, having a look at current [helpers source code](https://github.com/scrive/elm-form/blob/master/src/Form/Input.elm) should give you a good idea of the thing.

### Incremental validation

Similar to what Json.Extra provides you can also use `Form.andMap`

```elm
Form.succeed Player
    |> andMap (field "email" (string |> andThen email))
    |> andMap (field "power" int)
```

### Nested records

- Validation:

```elm
validation =
    map2 Player
        (field "email" (string |> andThen email))
        (field "power" (int |> andThen (minInt 0)))
        (field "options"
            (map2 Options
                (field "foo" string)
                (field "bar" string)
            )
        )
```

- View:

```elm
Input.textInput (Form.getFieldAsString "options.foo" form) []
```

### Dynamic lists

```elm
-- model
type alias TodoList =
    { title : String
    , items : List String
    }

-- validation
validation : Validation () Issue
validation =
    map2 TodoList
        (field "title" string)
        (field "items" (list string))

-- view
formView : Form () Issue -> Html Form.Msg
formView form =
    div
        [ class "todo-list" ]
        [ Input.textInput
            (Form.getFieldAsString "title" form)
            [ placeholder "Title" ]
        , div [ class "items" ] <|
            List.map
                (itemView form)
                (Form.getListIndexes "items" form)
        , button
            [ class "add"
            , onClick (Form.Append "items")
            ]
            [ text "Add" ]
        ]

itemView : Form () Issue -> Int -> Html Form.Msg
itemView form i =
    div
        [ class "item" ]
        [ Input.textInput
            (Form.getFieldAsString ("items." ++ (String.fromInt i)) form)
            []
        , a
            [ class "remove"
            , onClick (Form.RemoveItem "items" i)
            ]
            [ text "Remove" ]
        ]
```

### Initial values and reset

- At form initialization:

```elm
import Form.Field as Field


initialFields : List ( String, Field )
initialFields =
    [ ( "power", Field.string "10" )
    , ( "options"
      , Field.group
            [ ( "foo", Field.string "blah" )
            , ( "bar", Field.string "meh" )
            ]
      )
    ]


initialForm : Form
initialForm =
    Form.initial initialFields validation
```

See `Form.Field` type for more options.

- On demand:

```elm
button [ onClick (Form.Reset initialFields) ] [ text "Reset" ]
```

_Note:_ To have programmatic control over any `input[type=text]`/`textarea` value, like reseting or changing the value, you must set the `value` attribute with `Maybe.withDefault "" state.value`, as seen [here](https://github.com/etaque/elm-form/pull/57/files#diff-bfb877e82b2c89b329fcda943a258611R50). There's a downside of doing this: if the user types too fast, the caret can go crazy.

More info: https://github.com/evancz/elm-html/pull/81#issuecomment-145676200

### Custom errors

```elm
type LocalError = Fatal | NotSoBad

validation : Validation LocalError Foo
validation =
    (field "foo" (string |> customError Fatal))

-- creates `Form.Error.CustomError Fatal`
```

### Async validation

This package doesn't provide anything special for async validation, but doesn't prevent you to do that either. As field values are accessible from `update` with `Form.getStringAt/getBoolAt`, you can process them as you need, trigger effects like an HTTP request, and then add any errors to the view by yourself.

Another way would be to enable dynamic validation reload, to make it dependant of an effect, as it's part of the form state. Please ping me if this feature would be useful to you.
