---
title: typeorm getMany(), getRawMany()
date: "2023-10-10T22:36:37.121Z"
template: "post"
draft: false
category: "typeorm"
tags:
  - "typeorm"

description: "typeorm getMany(), getRawMany()"
---

쿼리빌더를 사용할 때 쓰게되는 getMany()와 getRawMany()
이 둘의 차이를 알아보자.

우선 getMany()를 타고 들어가보면

```typescript
getMany(): Promise<Entity[]>;
    Gets entities returned by execution of generated query builder sql.
```

위처럼 실행된 쿼리빌더의 sql에서 반환된 엔티티를 가져온다고 되어있다.

소스코드를 보면

```typescript
async getMany(): Promise<Entity[]> {
        if (this.expressionMap.lockMode === "optimistic")
            throw new OptimisticLockCanNotBeUsedError()

        const results = await this.getRawAndEntities()
        return results.entities
    }
```

getMany는 getRawAndEntities 메서드를 통해 가져온 값에서 엔티티를 반환하는 것을 알 수 있다.

여기서 getRawAndEntities를 보면

```typescript
async getRawAndEntities<T = any>(): Promise<{
        entities: Entity[]
        raw: T[]
    }> {
        const queryRunner = this.obtainQueryRunner()
        let transactionStartedByUs: boolean = false
        try {
            // start transaction if it was enabled
            if (
                this.expressionMap.useTransaction === true &&
                queryRunner.isTransactionActive === false
            ) {
                await queryRunner.startTransaction()
                transactionStartedByUs = true
            }

            this.expressionMap.queryEntity = true
            const results = await this.executeEntitiesAndRawResults(queryRunner)

            // close transaction if we started it
            if (transactionStartedByUs) {
                await queryRunner.commitTransaction()
            }

            return results
        } catch (error) {
            // rollback transaction if we started it
            if (transactionStartedByUs) {
                try {
                    await queryRunner.rollbackTransaction()
                } catch (rollbackError) {}
            }
            throw error
        } finally {
            if (queryRunner !== this.queryRunner)
                // means we created our own query runner
                await queryRunner.release()
        }
    }
```

쿼리빌더에서 생성된 sql을 실행하고 `executeEntitiesAndRawResults()`에서의 반환값을 리턴하는 것을 알 수 있다.

`executeEntitiesAndRawResults()`로 들어가보자

```typescript
protected async executeEntitiesAndRawResults(
        queryRunner: QueryRunner,
    ): Promise<{ entities: Entity[]; raw: any[] }> {
        if (!this.expressionMap.mainAlias)
            throw new TypeORMError(
                `Alias is not set. Use "from" method to set an alias.`,
            )

        if (
            (this.expressionMap.lockMode === "pessimistic_read" ||
                this.expressionMap.lockMode === "pessimistic_write" ||
                this.expressionMap.lockMode === "pessimistic_partial_write" ||
                this.expressionMap.lockMode === "pessimistic_write_or_fail" ||
                this.expressionMap.lockMode === "for_no_key_update" ||
                this.expressionMap.lockMode === "for_key_share") &&
            !queryRunner.isTransactionActive
        )
            throw new PessimisticLockTransactionRequiredError()

        if (this.expressionMap.lockMode === "optimistic") {
            const metadata = this.expressionMap.mainAlias.metadata
            if (!metadata.versionColumn && !metadata.updateDateColumn)
                throw new NoVersionOrUpdateDateColumnError(metadata.name)
        }

        const relationIdLoader = new RelationIdLoader(
            this.connection,
            queryRunner,
            this.expressionMap.relationIdAttributes,
        )
        const relationCountLoader = new RelationCountLoader(
            this.connection,
            queryRunner,
            this.expressionMap.relationCountAttributes,
        )
        const relationIdMetadataTransformer =
            new RelationIdMetadataToAttributeTransformer(this.expressionMap)
        relationIdMetadataTransformer.transform()
        const relationCountMetadataTransformer =
            new RelationCountMetadataToAttributeTransformer(this.expressionMap)
        relationCountMetadataTransformer.transform()

        let rawResults: any[] = [],
            entities: any[] = []

        // for pagination enabled (e.g. skip and take) its much more complicated - its a special process
        // where we make two queries to find the data we need
        // first query find ids in skip and take range
        // and second query loads the actual data in given ids range
        if (
            (this.expressionMap.skip || this.expressionMap.take) &&
            this.expressionMap.joinAttributes.length > 0
        ) {
            // we are skipping order by here because its not working in subqueries anyway
            // to make order by working we need to apply it on a distinct query
            const [selects, orderBys] =
                this.createOrderByCombinedWithSelectExpression("distinctAlias")
            const metadata = this.expressionMap.mainAlias.metadata
            const mainAliasName = this.expressionMap.mainAlias.name

            const querySelects = metadata.primaryColumns.map(
                (primaryColumn) => {
                    const distinctAlias = this.escape("distinctAlias")
                    const columnAlias = this.escape(
                        DriverUtils.buildAlias(
                            this.connection.driver,
                            undefined,
                            mainAliasName,
                            primaryColumn.databaseName,
                        ),
                    )
                    if (!orderBys[columnAlias])
                        // make sure we aren't overriding user-defined order in inverse direction
                        orderBys[columnAlias] = "ASC"

                    const alias = DriverUtils.buildAlias(
                        this.connection.driver,
                        undefined,
                        "ids_" + mainAliasName,
                        primaryColumn.databaseName,
                    )

                    return `${distinctAlias}.${columnAlias} AS ${this.escape(
                        alias,
                    )}`
                },
            )

            const originalQuery = this.clone()

            // preserve original timeTravel value since we set it to "false" in subquery
            const originalQueryTimeTravel =
                originalQuery.expressionMap.timeTravel

            rawResults = await new SelectQueryBuilder(
                this.connection,
                queryRunner,
            )
                .select(`DISTINCT ${querySelects.join(", ")}`)
                .addSelect(selects)
                .from(
                    `(${originalQuery
                        .orderBy()
                        .timeTravelQuery(false) // set it to "false" since time travel clause must appear at the very end and applies to the entire SELECT clause.
                        .getQuery()})`,
                    "distinctAlias",
                )
                .timeTravelQuery(originalQueryTimeTravel)
                .offset(this.expressionMap.skip)
                .limit(this.expressionMap.take)
                .orderBy(orderBys)
                .cache(
                    this.expressionMap.cache && this.expressionMap.cacheId
                        ? `${this.expressionMap.cacheId}-pagination`
                        : this.expressionMap.cache,
                    this.expressionMap.cacheDuration,
                )
                .setParameters(this.getParameters())
                .setNativeParameters(this.expressionMap.nativeParameters)
                .getRawMany()

            if (rawResults.length > 0) {
                let condition = ""
                const parameters: ObjectLiteral = {}
                if (metadata.hasMultiplePrimaryKeys) {
                    condition = rawResults
                        .map((result, index) => {
                            return metadata.primaryColumns
                                .map((primaryColumn) => {
                                    const paramKey = `orm_distinct_ids_${index}_${primaryColumn.databaseName}`
                                    const paramKeyResult =
                                        DriverUtils.buildAlias(
                                            this.connection.driver,
                                            undefined,
                                            "ids_" + mainAliasName,
                                            primaryColumn.databaseName,
                                        )
                                    parameters[paramKey] =
                                        result[paramKeyResult]
                                    return `${mainAliasName}.${primaryColumn.propertyPath}=:${paramKey}`
                                })
                                .join(" AND ")
                        })
                        .join(" OR ")
                } else {
                    const alias = DriverUtils.buildAlias(
                        this.connection.driver,
                        undefined,
                        "ids_" + mainAliasName,
                        metadata.primaryColumns[0].databaseName,
                    )

                    const ids = rawResults.map((result) => result[alias])
                    const areAllNumbers = ids.every(
                        (id: any) => typeof id === "number",
                    )
                    if (areAllNumbers) {
                        // fixes #190. if all numbers then its safe to perform query without parameter
                        condition = `${mainAliasName}.${
                            metadata.primaryColumns[0].propertyPath
                        } IN (${ids.join(", ")})`
                    } else {
                        parameters["orm_distinct_ids"] = ids
                        condition =
                            mainAliasName +
                            "." +
                            metadata.primaryColumns[0].propertyPath +
                            " IN (:...orm_distinct_ids)"
                    }
                }
                rawResults = await this.clone()
                    .mergeExpressionMap({
                        extraAppendedAndWhereCondition: condition,
                    })
                    .setParameters(parameters)
                    .loadRawResults(queryRunner)
            }
        } else {
            rawResults = await this.loadRawResults(queryRunner)
        }

        if (rawResults.length > 0) {
            // transform raw results into entities
            const rawRelationIdResults = await relationIdLoader.load(rawResults)
            const rawRelationCountResults = await relationCountLoader.load(
                rawResults,
            )
            const transformer = new RawSqlResultsToEntityTransformer(
                this.expressionMap,
                this.connection.driver,
                rawRelationIdResults,
                rawRelationCountResults,
                this.queryRunner,
            )
            entities = transformer.transform(
                rawResults,
                this.expressionMap.mainAlias!,
            )

            // broadcast all "after load" events
            if (
                this.expressionMap.callListeners === true &&
                this.expressionMap.mainAlias.hasMetadata
            ) {
                await queryRunner.broadcaster.broadcast(
                    "Load",
                    this.expressionMap.mainAlias.metadata,
                    entities,
                )
            }
        }

        if (this.expressionMap.relationLoadStrategy === "query") {
            const queryStrategyRelationIdLoader =
                new QueryStrategyRelationIdLoader(this.connection, queryRunner)

            await Promise.all(
                this.relationMetadatas.map(async (relation) => {
                    const relationTarget = relation.inverseEntityMetadata.target
                    const relationAlias =
                        relation.inverseEntityMetadata.targetName

                    const select = Array.isArray(this.findOptions.select)
                        ? OrmUtils.propertyPathsToTruthyObject(
                              this.findOptions.select as string[],
                          )
                        : this.findOptions.select
                    const relations = Array.isArray(this.findOptions.relations)
                        ? OrmUtils.propertyPathsToTruthyObject(
                              this.findOptions.relations,
                          )
                        : this.findOptions.relations

                    const queryBuilder = this.createQueryBuilder(queryRunner)
                        .select(relationAlias)
                        .from(relationTarget, relationAlias)
                        .setFindOptions({
                            select: select
                                ? OrmUtils.deepValue(
                                      select,
                                      relation.propertyPath,
                                  )
                                : undefined,
                            order: this.findOptions.order
                                ? OrmUtils.deepValue(
                                      this.findOptions.order,
                                      relation.propertyPath,
                                  )
                                : undefined,
                            relations: relations
                                ? OrmUtils.deepValue(
                                      relations,
                                      relation.propertyPath,
                                  )
                                : undefined,
                            withDeleted: this.findOptions.withDeleted,
                            relationLoadStrategy:
                                this.findOptions.relationLoadStrategy,
                        })
                    if (entities.length > 0) {
                        const relatedEntityGroups: any[] =
                            await queryStrategyRelationIdLoader.loadManyToManyRelationIdsAndGroup(
                                relation,
                                entities,
                                undefined,
                                queryBuilder,
                            )
                        entities.forEach((entity) => {
                            const relatedEntityGroup = relatedEntityGroups.find(
                                (group) => group.entity === entity,
                            )
                            if (relatedEntityGroup) {
                                const value =
                                    relatedEntityGroup.related === undefined
                                        ? null
                                        : relatedEntityGroup.related
                                relation.setEntityValue(entity, value)
                            }
                        })
                    }
                }),
            )
        }

        return {
            raw: rawResults,
            entities: entities,
        }
    }
```

굉장히 긴 코드가 나온다.

위 코드는 우선 lock에 대한 예외처리가 되어 있고, `loadRawResults`에서 원시결과를 가져오는 것을 알 수 있다.
그리고 가져온 원시결과를 `RawSqlResultsToEntityTransformer`클래스를 통해 엔티티로 변환하는 작업을 수행한다.
마지막에는 관계에 따른 로딩에 관련된 내용이 나오고,
원시결과와 엔티티를 반환하는 것을 알 수 있다.
위를 통해 원시데이터를 가져와 엔티티화 시키는 것을 알 수 있었다.

그럼 원시결과가 아닌 엔티티를 반환하는 `getMany()`에서의 특징은 무엇이 있을까?

```typescript
const queryBuilder = this.serviceRepository
  .createQueryBuilder("service")
  .select("service", "mapper")
  .leftJoin("service.categoryMapper", "mapper")
  .where("service.status = :status", { status: "Applying" })
  .getMany();
```

위 코드는 service 테이블에서 mapper 테이블을 조인한 코드이다.
`select`를 하지 않으면 조인을 했어도 기존(service) 엔티티만을 반환한다.

```typescript
// 결과값
// [
//   {...service entity, ...mapper entity}...
// ]
```

위의 결과를 받을 수 있다.

여기서 추가적으로 mapper entity에서의 일부 값을 가져온다고 했을 때
즉, `select('service','mapper.maincategoryid')` 이런식으로 가져오게 된다면
결과값에서 mapper entity 는 {maincategoryid : xx}가 될 것이다.

여기서 mapper entity의 다른 값에 접근을 하게 된다면 undefined가 나오게 된다.

이제 `getRawMany()`를 봐보면

```typescript
/**
     * Gets all raw results returned by execution of generated query builder sql.
     */
    getRawMany<T = any>(): Promise<T[]>;
```

`getMany`와 다르게 엔티티가 아닌 raw results 즉 원시결과를 가져오는 메서드이다.

소스코드를 보면

```typescript
async getRawMany<T = any>(): Promise<T[]> {
        if (this.expressionMap.lockMode === "optimistic")
            throw new OptimisticLockCanNotBeUsedError()

        this.expressionMap.queryEntity = false
        const queryRunner = this.obtainQueryRunner()
        let transactionStartedByUs: boolean = false
        try {
            // start transaction if it was enabled
            if (
                this.expressionMap.useTransaction === true &&
                queryRunner.isTransactionActive === false
            ) {
                await queryRunner.startTransaction()
                transactionStartedByUs = true
            }

            const results = await this.loadRawResults(queryRunner)

            // close transaction if we started it
            if (transactionStartedByUs) {
                await queryRunner.commitTransaction()
            }

            return results
        } catch (error) {
            // rollback transaction if we started it
            if (transactionStartedByUs) {
                try {
                    await queryRunner.rollbackTransaction()
                } catch (rollbackError) {}
            }
            throw error
        } finally {
            if (queryRunner !== this.queryRunner) {
                // means we created our own query runner
                await queryRunner.release()
            }
        }
    }
```

트랜잭션으로 `loadRawResults`가 감싸져 있고 `loadRawResults`를 통해 가져온 원시결과를 반환하는 것을 알 수 있다.
`loadRawResults`는 원시 결과를 가져오는 메서드이므로 `getRawMany`는 쿼리 결과를 순수한 JSON 객체의 배열로 반환한다.

원시결과를 반환하는 `getRawMany`에는 어떤 특징이 있을까?

```typescript
const queryBuilder = this.serviceRepository
  .createQueryBuilder("service")
  .select("service", "mapper")
  .leftJoin("service.categoryMapper", "mapper")
  .where("service.status = :status", { status: "Applying" })
  .getRawMany();
```

아까와 같은 service entity에서 mapper entity를 조인한 결과를 가져오는 쿼리빌더이다.
`getRawMany`는 JSON 객체이기 때문에 반환값에 각각의 alias가 붙어있다.

```typescript
{
  service_xx,
  service_xxx,
  mapper_xx,
  mapper_xxx,
}
```

`getMany`와 다른 점은 엔티티화 되어 있지 않기 때문에 조인한 테이블도 위처럼 하나의 객체안에 들어있고,
원시결과이기 때문에 공식문서에도 나와 있듯이 데이터를 통한 합계를 내거나 하는 작업을 할 수 있다.
But sometimes you need to select some specific data, let's say the sum of all user photos. This data is not an entity, it's called raw data. To get raw data, you use getRawOne and getRawMany.
예를 들자면

```typescript
ex.select("AVG(table.int) as avgStarPoint");
select(`IF(조건 IS NOT NULL, true, false) AS alias`);
select("SUM(user.photosCount)", "sum");
```

이러한 작업들을 할 수 있다고 볼 수 있다.

또한 `getRawMany`를 사용 할 때 `select('entity.*')`을 사용하면
entity_xx 이렇게 나오는 것을 xx 이렇게 나오게 할 수 있다.

그리고 원시결과에 접근하는 `getRawMany`는 결과값을 잘라낼 때 (ex . 페이지네이션)
skip, take가 아니라 offset, limit을 사용해야 한다.
skip과 take를 사용하면 적용이 되지 않아 전체가 나오는 것을 볼 수 있다.

순수하게 엔티티만 반환하는 api에서는 `getMany`가 보다 더 깔끔하게 보여줄 수 있고,
순수한 데이터가 필요한 경우거나 특정 열만 선택하는 경우 `getRawMany`가 보다 유용하다.
