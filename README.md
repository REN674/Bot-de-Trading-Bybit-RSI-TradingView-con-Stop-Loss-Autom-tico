# Bot de Trading Bybit RSI TradingView con Stop Loss Automático
**Última actualización:** 2025-04-29 02:57:44 UTC  
**Desarrollado por:** REN674

## 📊 Descripción
Bot de trading automatizado para Bybit que utiliza el RSI de TradingView para operar en el mercado de futuros. Incluye sistema de stop loss automático, trailing stop y modo hedge. El bot espera a que cada operación se cierre (por take profit o stop loss) antes de buscar nuevas entradas.

## ⭐ Características Principales
- 📈 RSI calculado igual que TradingView
- 🛡️ Stop Loss automático con trailing
- ⚔️ Modo Hedge (posiciones long/short simultáneas)
- 🔄 Gestión automática de operaciones
- ⏱️ Múltiples temporalidades
- 🔒 Sistema de seguridad para operaciones

## 🔧 Requisitos
- Python 3.8+
- Cuenta en Bybit
- API Key y Secret de Bybit
- Modo Hedge activado
- Dependencias:
  - pybit
  - pandas
  - numpy

## 🚀 Instalación

1. **Instalar dependencias:**
```bash
pip install pybit pandas numpy
