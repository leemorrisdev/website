+++
date = "2016-01-18T17:18:25Z"
tags = ["Development"]
title = "One-to-many with JDBI"

+++

I've been enjoying [JDBI](http://jdbi.org/) whilst working with [Dropwizard](http://www.dropwizard.io/0.9.1/docs/) - it's actually quite refreshing to hand-write the SQL that will hit the database.  I feel like the resulting application is considerably leaner in terms of data access, with the trade-off that obviously you need to spend time writing and mapping SQL to your POJOs.

On the whole it's been a positive experience, but I ran into an issue today that had be scratching my head for a short while.

**tl;dr - JDBI won't give you the full resultset when using a join if your API return type is not a collection.  To work around it, you need to fetch a collection and get the first result.**

Assume I have the following tables (columns removed for brevity):

    oms_order
    ============
    id              bigint
    stock           varchar(255)
    quantity        int(11)
    price           double
    quantity_filled int(11)
    
    oms_trade
    ============
    id              bigint
    quantity        int(11)
    price           double
    timestamp       datetime
    order_id        bigint
    
    public class Order {
    
        private long id;
        private String stock;
        private int quantity;
        private double price;
        private int quantityFilled = 0;
        
        private Set<Trade> trades;
        
    }
    
    public class Trade {
        
        private long id;
        private int quantity;
        private double price;
        private Date timestamp;
        
    }
    
The relationship is a simple one-to-many - one order can have multiple trades against it.  If I want to write a query to pick up the order and associated trades for a given order (in this case order 146), I can do this as follows:

    SELECT o.*, t.id as 'trade_id', t.quantity as 'trade_quantity', t.price as 'trade_price', t.timestamp as 'trade_ts' FROM ekuru.oms_order o left outer join ekuru.oms_trade t on o.id = t.order_id where o.id=146;

This will return me a result set such as :

    id    stock    quantity    price  quantity_filled   trade_id    trade_quantity    trade_price    trade_ts
    ======================================================================================================================
    146   VOD      50000       5.44   1200              1           500               5.43           2016-01-18 16:17:00
    146   VOD      50000       5.44   1200              2           700               5.42           2016-01-18 16:17:01
    
So far so good.  In terms of JDBI, I have the following in my DAO:

    @SqlQuery("select o.*, t.id as 'trade_id', t.quantity as 'trade_quantity', t.price as 'trade_price', t.timestamp as 'trade_ts' " +
              " from oms_order o left outer join oms_trade t on o.id = t.order_id where o.id = :id")
    public abstract Order findById(@Bind("id") Long id);
    
I've simply wrapped the SQL into a SqlQuery annotation, and provide the ID as a parameter.  Inside my mapper class, I want to read the order details off the first row and keep those as I know the following rows are duplicates.  I then want to (if they exist) read the trades by looping over the resultset and adding those to the trade collection.

    @Override
    public Order map(int index, ResultSet resultSet, StatementContext statementContext) throws SQLException {
    
        Order order = new Order();
        
        /*
         * Order details
         * /
        order.setId(resultSet.getLong("id"));
        order.setStock(resultSet.getString("stock"));
        order.setQuantity(resultSet.getInt("quantity"));
        order.setPrice(resultSet.getDouble("price"));
        order.setQuantityFilled(resultSet.getInt("quantity_filled");
        
        Set<Trade> trades = Sets.newHashSet();
        
        /*
         *  Only bother to gather trades if some of the order is filled
         */
        if(order.getQuantityFilled() > 0) {
        
            try {
            
                do {
                    if(resultSet.findColumn("trade_id") > 0 && resultSet.getLong("trade_id") > 0) {
                        Trade trade = new Trade();
                        trade.setId(resultSet.getLong("trade_id"));
                        trade.setQuantity(resultSet.getInt("trade_quantity"));
                        trade.setPrice(resultSet.getDouble("trade_price"));
                        trade.setTimestamp(resultSet.getDate("trade_ts"));

                        trades.add(trade);
                    }
                } while (resultSet.next());
                
            } catch (Exception ex) {
                // resultSet throws exceptions if you try to reference a missing field
            }
        }
        
        order.setTrades(trades);
        
        return order;
    }
    
This code runs fine, but there's a problem.  The REST endpoint attached to this would only show me one trade, even though I know I've got two in the database.  After some time looking at the SQL and a bit of googling, I found that JDBI will only give you access to a single row if your DAO return type is not a collection.

I'm not sure of the reasons behind this, I expected that my mapper would be presented with the full resultset.  To get around this is straight forward but is a bit smelly, so I'll be looking for a better solution.  We simply need to hide the annotated method, call it something else and create another findById method:

    public Order findById(@Bind("id") Long id) {
    
        List<Order> orders = findOne(id);
        
        if(orders.isEmpty()) {
            return null;
        }
        
        return orders.get(0);
    }
    
    @SqlQuery("select o.*, t.id as 'trade_id', t.quantity as 'trade_quantity', t.price as 'trade_price', t.timestamp as 'trade_ts' " +
              " from oms_order o left outer join oms_trade t on o.id = t.order_id where o.id = :id")
    public abstract List<Order> findOne(@Bind("id") Long id);
    
In this case the API still shows findById but delegates the actual work to findOne.  It's *not* a nice solution, but it works for now until I can figure out why JDBI doesn't support this out of the box.
