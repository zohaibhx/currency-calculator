# currency-calculator
from flask import Flask, request, render_template_string, redirect, url_for
import requests
from decimal import Decimal, InvalidOperation

app = Flask(__name__)

# Minimal static fallback rates (base: USD)
FALLBACK_RATES = {
    'USD': 1.0,
    'EUR': 0.92,
    'GBP': 0.79,
    'AED': 3.67,
    'INR': 83.50,
    'JPY': 156.40,
    'AUD': 1.56,
    'CAD': 1.35,
}

SUPPORTED_CURRENCIES = sorted(list(FALLBACK_RATES.keys()))

def fetch_rate_online(base: str, symbol: str):
    """Try to fetch a direct rate from exchangerate.host. Returns rate as Decimal or raises."""
    try:
        url = 'https://api.exchangerate.host/latest'
        params = {'base': base, 'symbols': symbol}
        resp = requests.get(url, params=params, timeout=5)
        resp.raise_for_status()
        data = resp.json()
        # data example: {'motd':..., 'success': True, 'base': 'USD', 'rates': {'EUR': 0.92}, 'date': '2025-10-25'}
        if not data.get('success', True):
            raise RuntimeError('API returned success=false')
        rate = data['rates'].get(symbol)
        if rate is None:
            raise KeyError('symbol not in response')
        return Decimal(str(rate))
    except Exception as e:
        raise


def get_rate(base: str, symbol: str):
    """Get exchange rate from base -> symbol. Returns (rate Decimal, used_fallback bool)."""
    
    if base == symbol:
        return Decimal('1'), False

    # try online
    try:
        rate = fetch_rate_online(base, symbol)
        return rate, False
    except Exception:
        # fallback: convert via USD fallback table (base USD)
        try:
            base_to_usd = Decimal(str(1.0)) / Decimal(str(FALLBACK_RATES[base])) if base in FALLBACK_RATES else None
            if base_to_usd is None:
                raise KeyError
            usd_to_target = Decimal(str(FALLBACK_RATES[symbol])) if symbol in FALLBACK_RATES else None
            if usd_to_target is None:
                raise KeyError
            rate = base_to_usd * usd_to_target
            return rate, True
        except Exception:
            # as a last resort, raise
            raise RuntimeError('Unable to determine exchange rate for %s -> %s' % (base, symbol))


@app.route('/', methods=['GET', 'POST'])
def index():
    error = None
    result = None
    rate = None
    used_fallback = False
    amount = ''
    from_curr = 'USD'
    to_curr = 'EUR'

    if request.method == 'POST':
        amount = request.form.get('amount', '').strip()
        from_curr = request.form.get('from_curr', from_curr).upper().strip()
        to_curr = request.form.get('to_curr', to_curr).upper().strip()

        # Basic validations
        try:
            dec_amount = Decimal(amount)
        except (InvalidOperation, ValueError):
            error = 'Enter a valid numeric amount (e.g. 123.45).'
            dec_amount = None

        if not error:
            if from_curr == '' or to_curr == '':
                error = 'Select both currencies.'
            else:
                try:
                    rate, used_fallback = get_rate(from_curr, to_curr)
                    converted = dec_amount * rate
                    # round to 6 decimal places sensibly
                    converted = converted.quantize(Decimal('0.000001'))
                    result = converted.normalize()
                    # keep rate as string
                    rate = rate.quantize(Decimal('0.000001')).normalize()
                except Exception as e:
                    error = 'Conversion failed: %s' % str(e)

    return render_template_string(TEMPLATE,
                                  currencies=SUPPORTED_CURRENCIES,
                                  error=error,
                                  result=result,
                                  rate=rate,
                                  amount=amount,
                                  from_curr=from_curr,
                                  to_curr=to_curr,
                                  used_fallback=used_fallback)


if __name__ == '__main__':
    # debug True for local development only
    app.run(host='127.0.0.1', port=5000, debug=True)
