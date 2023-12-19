**(Portuguese)**

**Lançando e capturando exceções de forma elegante**

Do contrário de um "Erro" que é algo imprevisível, e que não pode ser tratado, uma "Exceção" é algo que pode ser tratado
e previsto, e que pode ser lançado e capturado em qualquer lugar do código (JAVATPOINT, 2023). Tendo controle de onde e
quando a exceção é lançada, o desenvolvimento e gerenciamento do código se torna mais fácil e ágil.

O JavaScript não possui um mecanismo nativo para lançar e capturar exceções, mas é possível criar um mecanismo de
exceções utilizando o objeto `Error` e a palavra-chave `throw` (MDN, 2023), dessa forma, é possível não só lançar
exceções, mas também capturá-las e tratá-las de forma elegante.

Para conseguir capturar e tratar as exceções, é necessário encapsular o código que pode lançar uma exceção em um bloco,
e para isso, vamos utilizar o "Either", que é um tipo que pode ser um tipo ou outro, dessa forma, é possível criar um
tipo que representa o retorno de uma função, que pode ser um retorno de sucesso ou de erro. (HACKAGE, 2023)

```TypeScript
type BaseRequest<T, V> = (params?: T) => Promise<AxiosResponse<V>>;
```

Primeiro é necessário criar um tipo para a função que será utilizada para fazer as requisições, esse tipo deve receber
dois parâmetros, o primeiro é o tipo dos parâmetros que serão passados para a função, e o segundo é o tipo do retorno da
função. Nesse caso, o tipo dos parâmetros é genérico, e o tipo do retorno é uma `Promise` que retorna
um `AxiosResponse`, isso pode ser alterado de acordo com a necessidade ou biblioteca utilizada.

```TypeScript
type SuccessResponse<V> = {
    success: true;
    data: V;
};

type ErrorResponse<E = AxiosError> = {
    success: false;
    data: E
};
```

Em seguida, é necessário criar dois tipos, um para o retorno de sucesso, e outro para o retorno de erro. O tipo de
retorno de sucesso deve receber o tipo do retorno da função, e o tipo de retorno de erro deve receber o tipo do erro que
será retornado pela função, ambos contendo a propriedade `success` que indica se a requisição foi bem sucedida ou não, e
a propriedade `data` que contém o retorno da função ou o erro retornado pela função.

```TypeScript
type BaseResponse<V, E> = SuccessResponse<V> | ErrorResponse<E>;
```

O próximo passo é criar um tipo que recebe os tipos de retorno de sucesso e de erro, dessa forma, é possível criar um
tipo que representa o retorno da função, que pode ser um retorno de sucesso ou de erro.

```TypeScript
const requestHandler = <T, V, E = AxiosError>(request: BaseRequest<T, V>) => async (params?: T): Promise<BaseResponse<V, E>> => {
    try {
        const {data} = await request(params);

        return {
            success: true,
            data,
        };
    } catch (error) {
        return {
            success: false,
            data: error
        };
    }
};
```

Por fim, é necessário criar uma função que recebe a função que será utilizada para fazer as requisições, encapsulando
todas as requisições feitas com essa função, dessa forma, é possível tratar os erros, sem precisar repetir o código de
tratamento em todas as requisições.

```TypeScript
interface User {
    id: number;
    name: string;
};

interface GetUserParams {
    id: number;
};

const getUser = requestHandler<GetUserParams, User>(({id}) => axios.get(`/api/users/${id}`));
```

Aqui em cima fica um exemplo de como utilizar a função `requestHandler`.

**TLDR**

- "Erro" é algo imprevisível, e que não pode ser tratado. "Exceção" é algo que pode ser tratado e previsto, e que pode
  ser lançado e capturado em qualquer lugar do código.
- O JavaScript não possui um mecanismo nativo para lançar e capturar exceções, mas é possível criar um mecanismo de
  exceções utilizando o objeto `Error` e a palavra-chave `throw`.
- É possível criar tipos para melhorar o tratamento de erros, utilizando o "Either", que é um tipo que pode ser um tipo
  ou outro, dessa forma, é possível
  criar um tipo que representa o retorno de uma função, que pode ser um retorno de sucesso ou de erro.

**(English)**

**Throwing and catching exceptions beautifully**

Unlike an "Error" which is something unpredictable, and which cannot be handled, an "Exception" is something that can be
handled and predicted, and that can be thrown and caught anywhere in the code (JAVATPOINT, 2023). Having control of
where and when the exception is thrown, the development and management of the code becomes easier and faster.

JavaScript does not have a native mechanism for throwing and catching exceptions, but it is possible to create an exception mechanism using the `Error` object and the `throw` keyword (MDN, 2023), this way, it is possible not only to throw exceptions, but also to catch and handle them elegantly.

To be able to catch and handle exceptions, it is necessary to encapsulate the code that can throw an exception in a block, and for that, we will use the "Either", which is a type that can be one type or another, this way, it is possible to create a type that represents the return of a function, which can be a return of success or error. (HACKAGE, 2023)

```TypeScript
type BaseRequest<T, V> = (params?: T) => Promise<AxiosResponse<V>>;
```

First it is necessary to create a type for the function that will be used to make the requests, this type must receive two parameters, the first is the type of the parameters that will be passed to the function, and the second is the type of the return of the function. In this case, the type of the parameters is generic, and the type of the return is a `Promise` that returns an `AxiosResponse`, this can be changed according to the need or library used.

```TypeScript
type SuccessResponse<V> = {
    success: true;
    data: V;
};

type ErrorResponse<E = AxiosError> = {
    success: false;
    data: E
};
```

Then, it is necessary to create two types, one for the success return, and another for the error return. The type of success return must receive the type of the return of the function, and the type of error return must receive the type of the error that will be returned by the function, both containing the `success` property that indicates whether the request was successful or not, and the `data` property that contains the return of the function or the error returned by the function.

```TypeScript
type BaseResponse<V, E> = SuccessResponse<V> | ErrorResponse<E>;
```

The next step is to create a type that receives the types of success return and error return, this way, it is possible to create a type that represents the return of the function, which can be a return of success or error.

```TypeScript
const requestHandler = <T, V, E = AxiosError>(request: BaseRequest<T, V>) => async (params?: T): Promise<BaseResponse<V, E>> => {
    try {
        const {data} = await request(params);

        return {
            success: true,
            data,
        };
    } catch (error) {
        return {
            success: false,
            data: error
        };
    }
};
```

Finally, it is necessary to create a function that receives the function that will be used to make the requests, encapsulating all requests made with this function, this way, it is possible to handle errors, without having to repeat the handling code in all requests.

```TypeScript
interface User {
    id: number;
    name: string;
};

interface GetUserParams {
    id: number;
};

const getUser = requestHandler<GetUserParams, User>(({id}) => axios.get(`/api/users/${id}`));
```

Above is an example of how to use the `requestHandler` function.

**TLDR**

- "Error" is something unpredictable, and which cannot be handled. "Exception" is something that can be handled and predicted, and that can be thrown and caught anywhere in the code.
- JavaScript does not have a native mechanism for throwing and catching exceptions, but it is possible to create an exception mechanism using the `Error` object and the `throw` keyword.
- It is possible to create types to improve error handling, using the "Either", which is a type that can be one type or another, this way, it is possible to create a type that represents the return of a function, which can be a return of success or error.

**SOURCE**
---

HACKAGE. Either. Available at: <https://hackage.haskell.org/package/base-4.19.0.0/docs/Data-Either.html>. Accessed on:
Dec. 18, 2023.

JAVATPOINT. Exception Vs Error in Java. Available
at: <https://www.javatpoint.com/difference-between-exception-and-error-in-java>. Accessed on: Dec. 18, 2023.

MDN. Error. Available at: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error>.
Accessed on: Dec. 18, 2023.
