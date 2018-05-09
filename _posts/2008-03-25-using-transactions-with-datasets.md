---
id: 4
title: Using Transactions With DataSets
date: 2008-03-25T04:42:00+00:00
guid: http://www.btburnett.com/?p=4
permalink: /2008/03/using-transactions-with-datasets.html
categories:
  - .NET Framework
---
One of the major shortcomings of the ADO.NET dataset system, in my opinion, is the lack of support for transactions. In .NET 2.0 this situation was somewhat addressed by the addition of the System.Transactions namespace. However, this namespace has two flaws when it comes to using it with a DataSet. First, if you open more than one database connection within the same TransactionScope, it will use the Distributed Transaction Coordinator (MSDTC) instead of more efficient and configuration free database transactions. Second, if you require backwards compatibility with SQL 2000, it will ALWAYS use the MSDTC even if you open just a single database connection. The first issue can be addressed by opening the connection in advance and passing it to each TableAdapter that you use. Note that you'll need to change the Connection property on the TableAdapter from Friend (internal) to Public in the DataSet designer if the DataSet is in a DLL.

The second issue is a bit trickier to solve. In this case, you also need to open a single database connection as before, but then use an old fashioned SqlTransaction instead of the System.Transactions namespace. Then both the SqlConnection and SqlTransaction need to be set on each TableAdapter. This can be done by adding a special helper function to the partial class for the TableAdapter, as shown below.

```vb
Namespace DataSetTableAdapters

    Partial Class TableAdapter

        Public Sub SetConnection(ByVal cn As SqlConnection, trans As SqlTransaction)
            Connection = cn
            For Each cmd As SqlCommand In Me.CommandCollection
                cmd.Transaction = trans
            Next
            Adapter.UpdateCommand.Transaction = trans
            Adapter.InsertCommand.Transaction = trans
            Adapter.DeleteCommand.Transaction = trans
        End Sub

    End Class

End Namespace
```
