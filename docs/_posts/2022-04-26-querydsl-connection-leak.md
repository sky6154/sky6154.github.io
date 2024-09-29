---
title:  "Querydsl의 transform과 connection leak"
date:   2022-04-26 13:23:17 +0900
categories:
  - java
tags:
  - jpa
  - mysql
  - querydsl
excerpt: "Transform and connection leak in Querydsl"
toc: true
toc_sticky: true
---
## 개요
coupon을 개발중이었는데, 개발자 테스트 이후 개발환경에 배포되어 테스트를 해보니 오류와 함께 application이 동작하지 않았다.    
```
CONNECTION-POOL-COUPON-ADMIN-SLAVE - Pool stats (total=5, active=5, idle=0, waiting=0)
```
통계쿼리가 최초 n회는 잘 동작하다가 이후에 idle DB connection이 없어서 계속 오류가 나고 있었다. 

## 확인
querydsl에서 다음과 같이 가져오는 로직이 있었다.    
{% highlight java %}
public Map<Integer, Coupon> getAllCoupon() {
    return queryFactory
        .from(coupon)
        .transform(
            groupBy(coupon.id).as(coupon));
}
{% endhighlight %}

querydsl은 transform이라는 간편한 함수를 제공해주는데, 이를 이용해서 select한 결과를 로직내에서 사용하기 쉽게 변환하고 있었다.

## 해결
### @Transactional(readonly=true)
select라서 별도의 처리를 하지 않았는데, @Transactional을 붙이지 않으면 connection을 반환하지 않고 계속 물고있는다..  
transform을 사용시에는 Transactional을 붙여주어야 한다.

entityManager를 close 하는 조건에 특정 method가 호출되어야 닫히는 조건이 있는데, 
transform에서 사용하는 메소드는 대상이 아니다.

{% highlight java %}
public abstract class SharedEntityManagerCreator {

	private static final Class<?>[] NO_ENTITY_MANAGER_INTERFACES = new Class<?>[0];

	private static final Map<Class<?>, Class<?>[]> cachedQueryInterfaces = new ConcurrentReferenceHashMap<>(4);

	private static final Set<String> transactionRequiringMethods = new HashSet<>(6);

	private static final Set<String> queryTerminatingMethods = new HashSet<>(9);

	static {
		transactionRequiringMethods.add("joinTransaction");
		transactionRequiringMethods.add("flush");
		transactionRequiringMethods.add("persist");
		transactionRequiringMethods.add("merge");
		transactionRequiringMethods.add("remove");
		transactionRequiringMethods.add("refresh");

		queryTerminatingMethods.add("execute");  // javax.persistence.StoredProcedureQuery.execute()
		queryTerminatingMethods.add("executeUpdate");  // javax.persistence.Query.executeUpdate()
		queryTerminatingMethods.add("getSingleResult");  // javax.persistence.Query.getSingleResult()
		queryTerminatingMethods.add("getResultStream");  // javax.persistence.Query.getResultStream()
		queryTerminatingMethods.add("getResultList");  // javax.persistence.Query.getResultList()
		queryTerminatingMethods.add("list");  // org.hibernate.query.Query.list()
		queryTerminatingMethods.add("stream");  // org.hibernate.query.Query.stream()
		queryTerminatingMethods.add("uniqueResult");  // org.hibernate.query.Query.uniqueResult()
		queryTerminatingMethods.add("uniqueResultOptional");  // org.hibernate.query.Query.uniqueResultOptional()
	}

    // ...
{% endhighlight %}

{% highlight java %}
public class GroupByMap<K,V> extends AbstractGroupByTransformer<K, Map<K,V>> {

    GroupByMap(Expression<K> key, Expression<?>... expressions) {
        super(key, expressions);
    }

    @Override
    public Map<K, V> transform(FetchableQuery<?,?> query) {
        Map<K, Group> groups = new LinkedHashMap<K, Group>();

        // create groups
        FactoryExpression<Tuple> expr = FactoryExpressionUtils.wrap(Projections.tuple(expressions));
        boolean hasGroups = false;
        for (Expression<?> e : expr.getArgs()) {
            hasGroups |= e instanceof GroupExpression;
        }
        if (hasGroups) {
            expr = withoutGroupExpressions(expr);
        }
        try (CloseableIterator<Tuple> iter = query.select(expr).iterate()) {
            while (iter.hasNext()) {
                @SuppressWarnings("unchecked") //This type is mandated by the key type
                K[] row = (K[]) iter.next().toArray();
                K groupId = row[0];
                GroupImpl group = (GroupImpl) groups.get(groupId);
                if (group == null) {
                    group = new GroupImpl(groupExpressions, maps);
                    groups.put(groupId, group);
                }
                group.add(row);
            }
        }

        // transform groups
        return transform(groups);

    }

    // ...
{% endhighlight %}

`queryTerminatingMethods` 에 transform 내부 동작인 `query.select(expr).iterate()`인 `iterate`가 없다.  
따라서 query가 계속 종료되지 않고 남아있게 된다.

`@Transactional`을 사용할 경우 트랜잭션이 끝나면(commit or rollback) connection을 반납하게끔 되어있다.  
method의 로직이 모두 처리되고 닫히면 transaction도 끝나고 connection을 반납하므로 이슈가 없게 되는 것이다. 

## connection leak 탐지
hikari CP에서 connection leak 발생 시 자동으로 감지를 해주는 옵션이 있다.  
overhead 때문인지 default는 disable(0) 처리 되어있는데 다음과 같은 옵션으로 켤 수 있다.
```
leak-detection-threshold: 2000 // 2000이 최소값, 이하는 disable
```