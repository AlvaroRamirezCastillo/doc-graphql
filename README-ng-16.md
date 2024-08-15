# Configurar Graphql en Angular

## Instalaciones

instalar los siguientes paquetes.

```bash
npm install apollo-angular@5 @apollo/client graphql
```

## Configuraciones

crear el archivo **graphql.module.ts**

```bash
src/
│
├── app/
│   ├── core/
│   │   ├── graphql/
│   │   │   ├── graphql.module.ts
│   │   └── app.module.ts
```

```typescript
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { InMemoryCache } from '@apollo/client/core';
import { APOLLO_OPTIONS, ApolloModule } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';

@NgModule({
  imports: [
    HttpClientModule,
    ApolloModule,
  ],
  providers: [
    {
      provide: APOLLO_OPTIONS,
      useFactory: (httpLink: HttpLink) => {
        return {
          cache: new InMemoryCache(),
          link: httpLink.create({
            uri: 'http://localhost:4000' //url de tu servicio graphql
          })
        };
      },
      deps: [HttpLink],
    }
  ]
})
export class GraphqlModule { }
```

El módulo **GraphqlModule** debe ser importado en el **AppModule**

## Como hacer una consulta al servcio de graphql

Primero armar la **query**

```typescript
import { gql } from 'apollo-angular';

export class Example {
    findAllFavorites(): void {
    const query = gql`
      query AllFavorites {
        favorites {
          name
          type
        }
      }
    `;
  }
}
```

Segundo lanzar el request al servidor graphql

```typescript
import { inject } from '@angular/core';
import { Apollo, gql } from 'apollo-angular';

export class Example {
  private readonly apollo = inject(Apollo);

  findAllFavorites(): void {
    const query = gql`
      query AllFavorites {
        favorites {
          name
          type
        }
      }
    `;

    this.apollo
      .watchQuery({
        query
      })
      .valueChanges
      .subscribe(({ loading, data }) => {
        console.log({ loading, data });
      });
  }
}
```

## Como hacer una consulta al servcio de graphql con parametros


```typescript
...
findFavorite() {
  const query = gql`
    query FindFavorite($name: String!) {
      favoriteByName(name: $name) {
        name
        type
      }
    }
  `;

  this.apollo
    .watchQuery({
      query,
      variables: {
        name: 'papa'
      }
    })
    .valueChanges
    .subscribe(({ loading, data }) => {
      console.log({ loading, data });
    });
}
```

en la propiedad **variables** van los argumentos del **query favoriteByName**

## Como hacer una escritura o actualización (mutation) en un servidor graphql

```typescript
createFavorite() {
  const mutation = gql`
    mutation CreateFavorite($favoriteInput: FavoriteInput) {
      createFavorite(favoriteInput: $favoriteInput) {
        name
      }
    }
  `;

  this.apollo
    .mutate({
      mutation,
      variables: {
        favoriteInput: {
          name: 'netflix',
          type: 'payment'
        }
      }
    })
    .subscribe(({ loading, data }) => {
      console.log({ loading, data });
    });
}
```

## Como manejar errores de forma global.

primero crear el archivo **error-link.ts**

```bash
src/
│
├── app/
│   ├── core/
│   │   ├── graphql/
│   │   │   ├── graphql.module.ts
│   │   │   ├── error-link.ts
│   │   └── app.module.ts
```

```typescript
import { onError } from '@apollo/client/link/error';

export const errorLink = onError(({ graphQLErrors, networkError, response, forward, operation }) => {
  if (graphQLErrors)
    graphQLErrors.map(({ message, locations, path }) =>
      console.log(`[GraphQL error]: Message: ${message}, Location: ${locations}, Path: ${path}`),
    );

  if (networkError) console.log(`[Network error]: ${networkError}`);
});
```

segundo añadir el errorLink al modulo de graphql **GraphqlModule** por medio de la funcion from del paquete @apollo/client/core

```typescript
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { InMemoryCache, from } from '@apollo/client/core';
import { APOLLO_OPTIONS, ApolloModule } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { errorLink } from './error-link';

@NgModule({
  imports: [
    HttpClientModule,
    ApolloModule,
  ],
  providers: [
    {
      provide: APOLLO_OPTIONS,
      useFactory: (httpLink: HttpLink) => {
        return {
          cache: new InMemoryCache(),
          link: from([
            errorLink,
            httpLink.create({
              uri: 'http://localhost:4000'
            })
          ])
        };
      },
      deps: [HttpLink],
    }
  ]
})
export class GraphqlModule { }
```

Nota: es importante que el errorLink sea el primero en el arreglo.

## Como añadir cabeceras 

primero crear el archivo **auth-link.ts**

```bash
src/
│
├── app/
│   ├── core/
│   │   ├── graphql/
│   │   │   ├── graphql.module.ts
│   │   │   ├── auth-link.ts
│   │   └── app.module.ts
```

```typescript
import { setContext } from '@apollo/client/link/context';

export const authLink = setContext((_operation, { headers }) => {
  const token = 'token';

  return {
    headers: {
      ...headers,
      Authorization: `Bearer ${token}`,
    },
  };
});
```

segundo añadir el authLink al modulo de graphql **GraphqlModule** por medio de la funcion from del paquete @apollo/client/core

```typescript
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { InMemoryCache, from } from '@apollo/client/core';
import { APOLLO_OPTIONS, ApolloModule } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { errorLink } from './error-link';
import { authLink } from './auth-link';

@NgModule({
  imports: [
    HttpClientModule,
    ApolloModule,
  ],
  providers: [
    {
      provide: APOLLO_OPTIONS,
      useFactory: (httpLink: HttpLink) => {
        return {
          cache: new InMemoryCache(),
          link: from([
            errorLink,
            authLink,
            httpLink.create({
              uri: 'http://localhost:4000'
            })
          ])
        };
      },
      deps: [HttpLink],
    }
  ]
})
export class GraphqlModule { }
```

## manejo de cache

### Configurar cache de manera global

```typescript
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { InMemoryCache, from, ApolloClientOptions } from '@apollo/client/core';
import { APOLLO_OPTIONS, ApolloModule } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { errorLink } from './error-link';
import { authLink } from './auth-link';

@NgModule({
  imports: [
    HttpClientModule,
    ApolloModule,
  ],
  providers: [
    {
      provide: APOLLO_OPTIONS,
      useFactory: (httpLink: HttpLink): ApolloClientOptions<any> => {
        return {
          cache: new InMemoryCache(),
          link: from([
            errorLink,
            authLink,
            httpLink.create({
              uri: 'http://localhost:4000'
            })
          ]),
          defaultOptions: {
            query: {
              fetchPolicy: 'no-cache'
            },
            watchQuery: {
              fetchPolicy: 'no-cache'
            }
          }
        };
      },
      deps: [HttpLink],
    }
  ]
})
export class GraphqlModule { }
```

### Configurar cache por petición.

```typescript
this.apollo
  .watchQuery({
    query,
    variables: {
      name: 'papa'
    },
    fetchPolicy: 'cache-only'
  })
  .valueChanges
  .subscribe({
    next: ({ loading, data }) => {
      console.log({ loading, data });
    }
  });
```

opciones de cache:

- cache-first: Utiliza la caché si existe, de lo contrario, realiza la consulta a la red.
- cache-and-network: Utiliza la caché primero, pero siempre realiza la consulta a la red.
- network-only: Siempre realiza la consulta a la red, ignorando la caché.
- no-cache: No almacena los resultados en la caché.
- cache-only: Solo utiliza la caché, no realiza consultas a la red.

### Actualización Manual de la Caché

En algunos casos, es posible que necesites actualizar la caché manualmente, especialmente después de realizar mutaciones. Esto se puede hacer usando la función update en una mutación o el método writeQuery de la caché.

```typescript
this.apollo.mutate({
  mutation: MY_MUTATION,
  update: (cache, { data }) => {
    const existingData: any = cache.readQuery({ query: MY_QUERY });
    const newData = { ...existingData, myField: data.myField };
    cache.writeQuery({
      query: MY_QUERY,
      data: newData,
    });
  },
}).subscribe();
```

### Optimistic UI (Interfaz Optimista)

Apollo también soporta interfaces optimistas, donde se actualiza la UI de manera instantánea antes de que el servidor confirme el cambio, y luego se actualiza con los datos reales.

```typescript
this.apollo.mutate({
  mutation: MY_MUTATION,
  optimisticResponse: {
    myField: 'valor temporal',
  },
  update: (cache, { data }) => {
    // Actualiza la caché con los datos reales
  },
}).subscribe();
```

## generar automaticamente los servicios graphql.

instalar 

```bash
npm i -D @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-operations @graphql-codegen/typescript-apollo-angular
```

En la raiz del proyecto crear el archivo **codegen.ts**

```typescript
import type { CodegenConfig } from '@graphql-codegen/cli'

const config: CodegenConfig = {
  schema: 'schema.graphql',
  generates: {
    'src/app/feature/favorite/service/graphql/favorite.graphql.ts': {
      plugins: ['typescript', 'typescript-operations', 'typescript-apollo-angular'],
      documents: 'src/app/feature/favorite/graphql/*.graphql'
    }
  }
}

export default config
```

En el package.json agregar el siguiente script

```bash
"generate:graphql": "graphql-codegen"
```

en la raiz del proyecto agregar el archivo **schema.graphql** contendrá el schema del servicio graphql

```graphql
type Favorite {
  name: String
  type: String
}

type Query {
  favorites: [Favorite!]!
  favoriteByName(name: String!): Favorite!
}

input FavoriteInput {
  name: String!
  type: String!
}

type Mutation {
  createFavorite(favoriteInput: FavoriteInput): Favorite!
}
```

Ejemplo de uso:

```bash
src/
│
├── app/
│   ├── feature/
│   │   ├── favorite/
│   │   │   ├── graphql/
│   │   │   │   ├── all-favorites.query.graphql
```

all-favorites.query.graphql

```graphql
query AllFavorites {
  favorites {
    name
    type
  }
}
```

ejecutar

```bash
npm run generate:graphql
```



```typescript
import { Apollo, APOLLO_OPTIONS } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { ApplicationConfig, inject } from '@angular/core';
import { ApolloClientOptions, InMemoryCache } from '@apollo/client/core';

const uri = 'http://localhost:4000/'; // <-- add the URL of the GraphQL server here
export function apolloOptionsFactory(): ApolloClientOptions<any> {
  const httpLink = inject(HttpLink);
  return {
    link: httpLink.create({ uri }),
    cache: new InMemoryCache(),
  };
}

export const graphqlProvider: ApplicationConfig['providers'] = [
  Apollo,
  {
    provide: APOLLO_OPTIONS,
    useFactory: apolloOptionsFactory,
  },
];
```
