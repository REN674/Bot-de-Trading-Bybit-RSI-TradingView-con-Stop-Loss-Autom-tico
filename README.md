import config
import time
from pybit.unified_trading import HTTP
from decimal import Decimal, ROUND_FLOOR
from datetime import datetime
import pandas as pd
import numpy as np

# Inicializar sesión con Bybit
session = HTTP(
    testnet=False,
    api_key=config.api_key,
    api_secret=config.api_secret
)

def log_mensaje(mensaje):
    """
    Registra un mensaje con marca de tiempo UTC
    """
    tiempo_utc = datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')
    print(f"[{tiempo_utc} UTC] {mensaje}")

def obtener_posiciones(symbol):
    """
    Obtiene las posiciones abiertas
    """
    try:
        positions = session.get_positions(
            category="linear",
            symbol=symbol
        )
        return positions['result']['list']
    except Exception as e:
        log_mensaje(f"Error al obtener posiciones: {e}")
        return []

def verificar_posicion_abierta(symbol, side):
    """
    Verifica si hay una posición abierta para el símbolo y dirección dados
    """
    try:
        positions = obtener_posiciones(symbol)
        for position in positions:
            if float(position['size']) > 0:
                position_side = position['side']
                if position_side == side:
                    return True
        return False
    except Exception as e:
        log_mensaje(f"Error al verificar posición: {e}")
        return True

def calcular_rsi_tradingview(symbol, intervalo="1", periodo=14):
    """
    Calcula el RSI usando el método de TradingView
    """
    try:
        kline = session.get_kline(
            category="linear",
            symbol=symbol,
            interval=intervalo,
            limit=500
        )
        
        if 'result' not in kline or 'list' not in kline['result']:
            log_mensaje("Error: No se pudieron obtener datos de velas")
            return None

        # Convertir datos a DataFrame
        df = pd.DataFrame([
            {
                'timestamp': float(candle[0]),
                'open': float(candle[1]),
                'high': float(candle[2]),
                'low': float(candle[3]),
                'close': float(candle[4]),
                'volume': float(candle[5]),
            }
            for candle in kline['result']['list']
        ])

        # Ordenar por timestamp
        df = df.sort_values('timestamp')

        # Calcular cambios
        df['change'] = df['close'].diff()
        
        # Separar ganancias y pérdidas
        df['gain'] = df['change'].apply(lambda x: x if x > 0 else 0)
        df['loss'] = df['change'].apply(lambda x: abs(x) if x < 0 else 0)

        # Calcular SMA de ganancias y pérdidas
        avg_gain = df['gain'].rolling(window=periodo).mean()
        avg_loss = df['loss'].rolling(window=periodo).mean()

        # Calcular RS y RSI
        rs = avg_gain / avg_loss
        rsi = 100 - (100 / (1 + rs))

        return round(rsi.iloc[-1], 2)

    except Exception as e:
        log_mensaje(f"Error en el cálculo del RSI: {e}")
        return None

def ajustar_precio(symbol, price):
    """
    Ajusta el precio según las reglas de Bybit
    """
    try:
        step = session.get_instruments_info(category="linear", symbol=symbol)
        tick_size = float(step['result']['list'][0]['priceFilter']['tickSize'])
        price_scale = int(step['result']['list'][0]['priceScale'])
        
        tick_dec = Decimal(str(tick_size))
        precio_final = Decimal(str(price)).quantize(Decimal("1." + "0" * price_scale), rounding=ROUND_FLOOR)
        operacion_dec = (precio_final / tick_dec).quantize(Decimal('1'), rounding=ROUND_FLOOR) * tick_dec
        
        return float(operacion_dec)
    except Exception as e:
        log_mensaje(f"Error al ajustar el precio: {e}")
        return price

def ajustar_cantidad(symbol, qty):
    """
    Ajusta la cantidad según las reglas de Bybit
    """
    try:
        step = session.get_instruments_info(category="linear", symbol=symbol)
        min_qty = float(step['result']['list'][0]['lotSizeFilter']['minOrderQty'])
        step_size = float(step['result']['list'][0]['lotSizeFilter']['qtyStep'])

        if qty < min_qty:
            qty = min_qty
        else:
            qty = (qty // step_size) * step_size

        return qty
    except Exception as e:
        log_mensaje(f"Error al ajustar la cantidad: {e}")
        return qty

def obtener_precio_actual(symbol):
    """
    Obtiene el precio actual del mercado
    """
    try:
        ticker = session.get_tickers(category="linear", symbol=symbol)
        precio_actual = float(ticker['result']['list'][0]['lastPrice'])
        log_mensaje(f"Precio actual de {symbol}: {precio_actual}")
        return precio_actual
    except Exception as e:
        log_mensaje(f"Error al obtener el precio actual: {e}")
        return None

def actualizar_stop_loss(symbol, side, entrada, precio_actual, stop_loss_inicial, trailing_step):
    """
    Actualiza el stop loss basado en el movimiento del precio
    """
    try:
        positions = obtener_posiciones(symbol)
        for position in positions:
            if float(position['size']) > 0:
                position_side = "Buy" if position['side'] == "Buy" else "Sell"
                if position_side == side:
                    entrada_precio = float(position['avgPrice'])
                    stop_loss_actual = float(position['stopLoss']) if position['stopLoss'] != '0' else None
                    
                    if side == "Buy":
                        profit_actual = (precio_actual - entrada_precio) / entrada_precio * 100
                        new_stop_loss = None
                        
                        if profit_actual >= trailing_step:
                            steps = int(profit_actual / trailing_step)
                            new_stop_loss = entrada_precio * (1 + (steps - 1) * trailing_step / 100)
                            new_stop_loss = max(new_stop_loss, stop_loss_actual if stop_loss_actual else entrada_precio * (1 - stop_loss_inicial / 100))
                    else:
                        profit_actual = (entrada_precio - precio_actual) / entrada_precio * 100
                        new_stop_loss = None
                        
                        if profit_actual >= trailing_step:
                            steps = int(profit_actual / trailing_step)
                            new_stop_loss = entrada_precio * (1 - (steps - 1) * trailing_step / 100)
                            new_stop_loss = min(new_stop_loss, stop_loss_actual if stop_loss_actual else entrada_precio * (1 + stop_loss_inicial / 100))
                    
                    if new_stop_loss and (stop_loss_actual is None or 
                        (side == "Buy" and new_stop_loss > stop_loss_actual) or 
                        (side == "Sell" and new_stop_loss < stop_loss_actual)):
                        
                        sl_tp_payload = {
                            "category": "linear",
                            "symbol": symbol,
                            "stopLoss": new_stop_loss,
                            "positionIdx": 1 if side == "Buy" else 2
                        }
                        
                        result = session.set_trading_stop(**sl_tp_payload)
                        log_mensaje(f"Stop Loss actualizado para {side}: {new_stop_loss}")
                        
    except Exception as e:
        log_mensaje(f"Error al actualizar stop loss: {e}")

def colocar_orden(symbol, side, capital_usdt, precio_entrada, stop_loss_pct, take_profit_pct):
    """
    Coloca una orden con soporte para modo hedge
    """
    try:
        qty = round(capital_usdt / precio_entrada, 6)
        qty = ajustar_cantidad(symbol, qty)
        precio_entrada = ajustar_precio(symbol, precio_entrada)

        if side == "Buy":
            stop_loss = precio_entrada * (1 - stop_loss_pct / 100)
            take_profit = precio_entrada * (1 + take_profit_pct / 100)
        else:
            stop_loss = precio_entrada * (1 + stop_loss_pct / 100)
            take_profit = precio_entrada * (1 - take_profit_pct / 100)

        stop_loss = ajustar_precio(symbol, stop_loss)
        take_profit = ajustar_precio(symbol, take_profit)

        position_idx = 1 if side == "Buy" else 2
        
        order_payload = {
            "category": "linear",
            "symbol": symbol,
            "side": side,
            "orderType": "Market",
            "qty": qty,
            "positionIdx": position_idx,
        }

        order = session.place_order(**order_payload)
        log_mensaje(f"Orden de entrada colocada: {order}")

        sl_tp_payload = {
            "category": "linear",
            "symbol": symbol,
            "stopLoss": stop_loss,
            "takeProfit": take_profit,
            "positionIdx": position_idx,
        }
        
        sl_tp_order = session.set_trading_stop(**sl_tp_payload)
        log_mensaje(f"Stop Loss y Take Profit configurados: {sl_tp_order}")

        return order
    except Exception as e:
        log_mensaje(f"Error al colocar la orden: {e}")
        return None

def ejecutar_bot():
    log_mensaje("=== BOT DE TRADING EN MODO HEDGE CON RSI (TRADINGVIEW) Y STOP LOSS AUTOMÁTICO ===")
    log_mensaje(f"Usuario: REN674")
    log_mensaje(f"Fecha y hora de inicio: {datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')} UTC")
    
    # Configuración inicial
    symbol = input("Ingrese el símbolo (por ejemplo, XRP): ").upper() + "USDT"
    capital_usdt = float(input("Ingrese el tamaño de la posición en USDT: "))
    stop_loss_pct = float(input("Ingrese el porcentaje de Stop Loss inicial (por ejemplo, 2): "))
    take_profit_pct = float(input("Ingrese el porcentaje de Take Profit (por ejemplo, 5): "))
    trailing_step = float(input("Ingrese el paso del trailing stop en porcentaje (por ejemplo, 0.5): "))
    rsi_long = float(input("Ingrese el valor de RSI para entrar en Long (por ejemplo, 30): "))
    rsi_short = float(input("Ingrese el valor de RSI para entrar en Short (por ejemplo, 70): "))
    intervalo = input("Ingrese la temporalidad para el cálculo del RSI (1, 3, 5, 15, 30, 60, etc.): ")

    posicion_actual = None

    while True:
        try:
            precio_actual = obtener_precio_actual(symbol)
            if precio_actual is None:
                continue

            # Si hay una posición abierta, solo actualizar stop loss y esperar
            if posicion_actual:
                actualizar_stop_loss(symbol, posicion_actual, None, precio_actual, stop_loss_pct, trailing_step)
                
                # Verificar si la posición sigue abierta
                if not verificar_posicion_abierta(symbol, posicion_actual):
                    log_mensaje(f"Posición {posicion_actual} cerrada. Reiniciando ciclo de trading.")
                    posicion_actual = None
                    time.sleep(60)
                    continue
                
                time.sleep(10)
                continue

            # Si no hay posición abierta, buscar nueva entrada
            rsi = calcular_rsi_tradingview(symbol, intervalo=intervalo)

            if rsi is not None:
                log_mensaje(f"RSI actual para {symbol}: {rsi}")

                if rsi < rsi_long and not verificar_posicion_abierta(symbol, "Buy"):
                    log_mensaje("Condición de entrada para Long detectada.")
                    orden = colocar_orden(symbol, "Buy", capital_usdt, precio_actual, stop_loss_pct, take_profit_pct)
                    if orden:
                        posicion_actual = "Buy"
                        log_mensaje("Esperando cierre de posición Long...")

                elif rsi > rsi_short and not verificar_posicion_abierta(symbol, "Sell"):
                    log_mensaje("Condición de entrada para Short detectada.")
                    orden = colocar_orden(symbol, "Sell", capital_usdt, precio_actual, stop_loss_pct, take_profit_pct)
                    if orden:
                        posicion_actual = "Sell"
                        log_mensaje("Esperando cierre de posición Short...")

            time.sleep(60)

        except Exception as e:
            log_mensaje(f"Error en el bot: {e}")
            time.sleep(10)

if __name__ == "__main__":
    ejecutar_bot()
