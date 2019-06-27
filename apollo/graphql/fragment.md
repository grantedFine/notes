### fragment重用
 graphql的前端document node解析如果每个地方都将要调取资源所有字段都声明一遍,既显繁琐也不利于迭代, 每一次资源的新字段添加都会导致所有前端使用到的document node更改, 如果使用 fragment , 则每次迭代只需要修改对应fragment, 大大减少工作量和犯错的可能性, fragment使用如下:

```
  const resourceResult = gql`
    fragment ResourceResult on Resource {
      id
      displayName
      name
      type
      owner
    }
  `

  const fetchResource: DocumentNode = gql`
    query getResource($id: ID!) {
      Resource(id: $id) {
        ...ResourceResult
      }
    }
    ${resourceResult}
  `
```
