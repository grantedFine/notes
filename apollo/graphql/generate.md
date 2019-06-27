### graphql的前端schema生成
apollo的后端下载方法可参考官方, 这里要说的是理论上后端的graphql定义完成后可以直接使用graphql文件生成schema,而不必等待后端建立service后再生成,这对于前后端同一个repo的开发来说非常节约时间

```
const { GraphQLTypesLoader } = require('@nestjs/graphql')
const { graphql, introspectionQuery } = require('graphql')
const { makeExecutableSchema } = require('graphql-tools')
const { join } = require('path')
const { writeFileSync } = require('fs')

async function genSchema(path) {
  const loader = new GraphQLTypesLoader()
  const typeDefs = loader.mergeTypesByPaths(...[join(__dirname, '../backend/src/**/*.graphql')])
  const schema = makeExecutableSchema({
    typeDefs,
  })

  const result = await graphql(schema, introspectionQuery)

  writeFileSync(`${path}.json`, JSON.stringify(result.data, null, 2))

  console.info('Generated GraphQL schema.json')
}

genSchema('./src/schema/schema')

```