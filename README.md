//+------------------------------------------------------------------+
//| Hull Suite Strategy - MQL4                                       |
//+------------------------------------------------------------------+
#property strict

input int Length = 55;
input double Lots = 0.1;
input int Slippage = 3;
input int Magic = 555;

//+------------------------------------------------------------------+
// WMA
//+------------------------------------------------------------------+
double WMA(int period,int shift)
{
   return iMA(NULL,0,period,0,MODE_LWMA,PRICE_CLOSE,shift);
}

//+------------------------------------------------------------------+
// HMA
//+------------------------------------------------------------------+
double HMA(int period,int shift)
{
   double wma1 = WMA(period/2,shift);
   double wma2 = WMA(period,shift);

   double raw = 2.0*wma1 - wma2;

   return raw;
}

//+------------------------------------------------------------------+
// POSICIÓN ACTUAL
//+------------------------------------------------------------------+
int CurrentPosition()
{
   for(int i=0;i<OrdersTotal();i++)
   {
      if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES))
      {
         if(OrderMagicNumber()==Magic)
         {
            if(OrderType()==OP_BUY) return 1;
            if(OrderType()==OP_SELL) return -1;
         }
      }
   }
   return 0;
}

//+------------------------------------------------------------------+
// CERRAR TODAS
//+------------------------------------------------------------------+
void CloseAll()
{
   for(int i=OrdersTotal()-1;i>=0;i--)
   {
      if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES))
      {
         if(OrderMagicNumber()!=Magic) continue;

         if(OrderType()==OP_BUY)
            OrderClose(OrderTicket(),OrderLots(),Bid,Slippage);

         if(OrderType()==OP_SELL)
            OrderClose(OrderTicket(),OrderLots(),Ask,Slippage);
      }
   }
}

//+------------------------------------------------------------------+
// ON TICK
//+------------------------------------------------------------------+
void OnTick()
{
   static datetime lastBar=0;

   if(Time[0]==lastBar) return;
   lastBar=Time[0];

   // HULL
   double hull0 = HMA(Length,0);
   double hull2 = HMA(Length,2);

   bool bullish = hull0 > hull2;
   bool bearish = hull0 < hull2;

   int pos = CurrentPosition();

   // BUY
   if(pos <= 0 && bullish)
   {
      CloseAll();

      OrderSend(Symbol(),
                OP_BUY,
                Lots,
                Ask,
                Slippage,
                0,
                0,
                "Hull Buy",
                Magic,
                0,
                clrGreen);
   }

   // SELL
   if(pos >= 0 && bearish)
   {
      CloseAll();

      OrderSend(Symbol(),
                OP_SELL,
                Lots,
                Bid,
                Slippage,
                0,
                0,
                "Hull Sell",
                Magic,
                0,
                clrRed);
   }
}
