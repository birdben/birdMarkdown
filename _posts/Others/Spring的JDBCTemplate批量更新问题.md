---
title: "Spring的JDBCTemplate批量更新的性能问题"
date: 2016-07-03 11:25:38
tags: [Spring]
categories: [Web框架]
---

这两天偶然发现了一个JDBCTemplate批量更新的性能问题，问题是这样的一次性批量删除并且插入3000条记录，居然用了400s我也是被吓到了。我也看了一下代码确实用的是jdbcTemplate的batchUpdate方法，用法都没有错，正常执行SQL批量更新3000条数据应该也是秒级的，但是不清楚为什么使用jdbcTemplate的性能会和预期差这么多呢，后来经过反复测试才发现原来是参数类型惹的祸，具体请看下面的代码片段。

### 修改前的jdbcTemplate.batchUpdate代码片段
```
public void batchUpdateAcl(List<Auth> authBeans) {
	List<Object[]> deleteParams = new ArrayList<Object[]>();
	List<Object[]> insertParams = new ArrayList<Object[]>();

	for (Auth auth : authBeans) {
		String resourceID = auth.getResourceID();
		String targetID = auth.getTargetID();
		String targetType = auth.getType() == null ? AuthTargetType.ROLE.name() : auth.getType().name();
		int acl = AuthEnum.formateAclCode(auth.getAcls());

		deleteParams.add(new Object[] { targetID, resourceID, targetType });
		insertParams.add(new Object[] { targetID, resourceID, targetType, acl });
	}

	String sql = "DELETE FROM " + TABLE_NAME + " WHERE target=? AND resource=? and targetType=?";
	// 批量删除修改前的调用方式
	jdbc.batchUpdate(sql, deleteParams);

	sql = "INSERT INTO " + TABLE_NAME + " (target, resource, targetType, acl) VALUES(?,?,?,?)";
	// 批量插入修改前的调用方式
	jdbc.batchUpdate(sql, insertParams);
}
```


### 修改后的jdbcTemplate.batchUpdate代码片段
```
public void batchUpdateAcl(List<Auth> authBeans) {
	final List<Object[]> deleteParams = new ArrayList<Object[]>();
	final List<Object[]> insertParams = new ArrayList<Object[]>();

	for (Auth auth : authBeans) {
		String resourceID = auth.getResourceID();
		String targetID = auth.getTargetID();
		String targetType = auth.getType() == null ? AuthTargetType.ROLE.name() : auth.getType().name();
		int acl = AuthEnum.formateAclCode(auth.getAcls());

		deleteParams.add(new Object[] { targetID, resourceID, targetType });
		insertParams.add(new Object[] { targetID, resourceID, targetType, acl });
	}

	String sql = "DELETE FROM " + TABLE_NAME + " WHERE target=? AND resource=? and targetType=?";
	// 批量删除修改后的调用方式
	jdbc.batchUpdate(sql, new BatchPreparedStatementSetter() {
		@Override
		public void setValues(PreparedStatement ps, int i) throws SQLException {
			Object[] args = deleteParams.get(i);
			ps.setString(1, (String) args[0]);
			ps.setString(2, (String) args[1]);
			ps.setString(3, (String) args[2]);
		}

		@Override
		public int getBatchSize() {
			return 0;
		}
	});

	sql = "INSERT INTO " + TABLE_NAME + " (target, resource, targetType, acl) VALUES(?,?,?,?)";
	// 批量插入修改后的调用方式
	jdbc.batchUpdate(sql, new BatchPreparedStatementSetter() {
		@Override
		public void setValues(PreparedStatement ps, int i) throws SQLException {
			Object[] args = insertParams.get(i);
			ps.setString(1, (String) args[0]);
			ps.setString(2, (String) args[1]);
			ps.setString(3, (String) args[2]);
			ps.setString(4, (String) args[3]);
		}

		@Override
		public int getBatchSize() {
			return 0;
		}
	});
}
```

具体调用的方式有些变化，原来的调用方式jdbc.batchUpdate(sql, deleteParams);的参数是List\<Object[]>类型，而修改后的调用方式是实现的new BatchPreparedStatementSetter()接口的方法，只是这里的区别，但是性能却相差甚远。
