---
id: 4
title: Using Transactions With DataSets
date: 2008-03-25T04:42:00+00:00
guid: http://www.btburnett.com/?p=4
permalink: /2008/03/using-transactions-with-datasets.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/03/using-transactions-with-datasets.html
categories:
  - ADO.NET
---
<span style="font-family:verdana;font-size:85%;">One of the major shortcomings of the ADO.NET dataset system, in my opinion, is the lack of support for transactions. In .NET 2.0 this situation was somewhat addressed by the addition of the System.Transactions namespace. However, this namespace has two flaws when it comes to using it with a DataSet. First, if you open more than one database connection within the same TransactionScope, it will use the Distributed Transaction Coordinator (MSDTC) instead of more efficient and configuration free database transactions. Second, if you require backwards compatibility with SQL 2000, it will ALWAYS use the MSDTC even if you open just a single database connection. The first issue can be addressed by opening the connection in advance and passing it to each TableAdapter that you use. Note that you&#8217;ll need to change the Connection property on the TableAdapter from Friend (internal) to Public in the DataSet designer if the DataSet is in a DLL.</span>
<span style="font-family:verdana;font-size:85%;"></span>
<span style="font-family:verdana;font-size:85%;">The second issue is a bit trickier to solve. In this case, you also need to open a single database connection as before, but then use an old fashioned SqlTransaction instead of the System.Transactions namespace. Then both the SqlConnection and SqlTransaction need to be set on each TableAdapter. This can be done by adding a special helper function to the partial class for the TableAdapter, as shown below.</span>

<span style="font-family:courier new;font-size:70%">Namespace DataSetTableAdapters</span>
<span style="font-family:Courier New;font-size:70%;"></span>
<span style="font-family:Courier New;font-size:70%;margin-left: 2em;">Partial Class TableAdapter</span>
<span style="font-family:Courier New;font-size:70%;"></span>
<span style="font-family:courier new;font-size:70%;margin-left: 4em;">Public Sub SetConnection(ByVal cn As SqlConnection, trans As SqlTransaction)</span>
<span style="font-family:courier new;font-size:70%;margin-left: 6em;">Connection = cn</span>
<span style="font-family:courier new;font-size:70%;margin-left: 6em;">For Each cmd As SqlCommand In Me.CommandCollection</span>
<span style="font-family:courier new;font-size:70%;margin-left: 8em;">cmd.Transaction = trans</span>
<span style="font-family:courier new;font-size:70%;margin-left: 6em;">Next</span>
<span style="font-family:courier new;font-size:70%;margin-left: 6em;">Adapter.UpdateCommand.Transaction = trans</span>
<span style="font-family:courier new;font-size:70%;margin-left: 6em;">Adapter.InsertCommand.Transaction = trans</span>
<span style="font-family:courier new;font-size:70%;margin-left: 6em;">Adapter.DeleteCommand.Transaction = trans</span>
<span style="font-family:courier new;font-size:70%;margin-left: 4em;">End Sub</span>
<span style="font-family:Courier New;font-size:70%;"></span>
<span style="font-family:Courier New;font-size:70%;margin-left: 2em;">End Class</span>
<span style="font-family:Courier New;font-size:70%;"></span>
<span style="font-family:Courier New;font-size:70%;">End Namespace</span>
