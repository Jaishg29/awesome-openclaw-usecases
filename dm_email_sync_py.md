import json, subprocess, re, os, base64, html, datetime
from email.utils import parsedate_to_datetime

ACCOUNT='[Enter Email used for parsing]'
SHEET='[Google sheet ID]'
BASE_FROM_QUERY='after:[Date form when to start parsing] -label:[logged label name] -label:[review label name]'
MAX_MESSAGES=25
USD_PER_INR=0.01197
USD_PER_CAD=0.74
USD_PER_GBP=1.25
USD_PER_EUR=1.08
CATEGORY_KEYWORDS = [
    ('Groceries',['whole foods','heb','costco','trader joe','grocery','blinkit','instamart','bigbasket','zepto']),
    ('Dining',['restaurant','cafe','coffee','starbucks','eat','dinner','lunch','breakfast','takeout','pizza','swiggy','zomato']),
    ('Shopping',['amazon','target','walmart','mall','shopping','clothes','zara','nike','lululemon','myntra','ajio']),
    ('Health',['doctor','pharmacy','cvs','walgreens','medicine','hospital','clinic','apollo pharmacy','dermatology']),
    ('Transportation',['uber','lyft','ola','gas','shell','bp','fuel','parking','toll','fastag','metro','train ride']),
    ('Travel',['flight','hotel','airbnb','air canada','united','budget rental','car rental','trip']),
    ('Pets',['chewy','vet','dog','pet','luna','jelly','petsmart','farmina','pet zone','petzone','marshalls pet zone']),
    ('Subscriptions',['netflix','spotify','hulu','apple one','chatgpt','google one','subscription','membership']),
    ('Utilities',['electricity','internet','water','phone','bill','utility']),
    ('Rent/Housing',['rent','mortgage','housing']),
    ('Entertainment',['movie','concert','ticket','game','streaming','theater']),
    ('Personal Care',['salon','spa','skincare','cosmetics','beauty','grooming']),
    ('Insurance',['insurance','premium']),
    ('Gifts',['gift','birthday gift','present']),
]
SUBCATEGORY_HINTS = [
    ('Takeout',['uber eats','door dash']),
    ('Dog Food',['chewy','farmina']),
    ('Pet Supplies',['pet zone','petsmart']),
    ('Lodging',['airbnb','hotel']),
    ('Car Rental',['budget rental','car rental']),
    ('Parking',['parking']),
    ('Rideshare',['uber','lyft','ola']),
]
MERCHANT_PATTERNS = [
    r'debited at\s+([A-Za-z0-9&.,\- ]+)',
    r'paid to\s+([A-Za-z0-9&.,\- ]+)',
    r'merchant[:\s]+([A-Za-z0-9&.,\- ]+)',
    r'at\s+([A-Za-z0-9&.,\- ]+?)\s+(?:on|for|via|using|from|ref|txn|transaction)',
]
DATE_PATTERNS = [
    r'(\b\d{1,2}[/-]\d{1,2}[/-]\d{2,4}\b)',
    r'(\b\d{1,2}\s+[A-Za-z]{3,9}\s+\d{2,4}\b)',
    r'(\b[A-Za-z]{3,9}\s+\d{1,2},\s*\d{4}\b)',
]
TIME_PATTERNS = [
    r'(\b\d{1,2}:\d{2}\s?[APMapm]{2}\b)',
    r'(\b\d{1,2}:\d{2}:\d{2}\b)',
]
AMOUNT_PATTERNS = [
    ('INR', r'(?:Rs\.?|INR|₹)\s*([0-9][0-9,]*\.?[0-9]{0,2})'),
    ('USD', r'\$\s*([0-9][0-9,]*\.?[0-9]{0,2})'),
    ('GBP', r'£\s*([0-9][0-9,]*\.?[0-9]{0,2})'),
    ('EUR', r'€\s*([0-9][0-9,]*\.?[0-9]{0,2})'),
    ('CAD', r'CAD\s*([0-9][0-9,]*\.?[0-9]{0,2})'),
]
RETAILER_MERCHANTS = {
    'amazon': 'Amazon',
    'allbirds': 'Allbirds',
    'lululemon': 'lululemon',
}
TRAVEL_MERCHANTS = {
    'airbnb': 'Airbnb',
    'budget': 'Budget',
    'united': 'United',
    'air canada': 'Air Canada',
}
DIRECT_MERCHANTS = {
    'chugh llp': 'Chugh LLP',
    'xpel': 'XPEL',
    'above and beyond': 'Above & Beyond',
    'frontgatetickets': 'Above & Beyond',
    'permit2park': 'City of Neptune Beach Parking',
    'toyota center': 'Toyota Center',
    'axs': 'Toyota Center',
    'tesla': 'Tesla',
}
def run(cmd):
    env=os.environ.copy()
    env['GOG_KEYRING_BACKEND']='file'
    p=subprocess.run(cmd, capture_output=True, text=True, env=env)
    if p.returncode!=0:
        raise RuntimeError(f"cmd failed: {' '.join(cmd)}\nSTDOUT:\n{p.stdout}\nSTDERR:\n{p.stderr}")
    return p.stdout
def gmail_search_messages(query, page_token=None):
    cmd=['gog','gmail','messages','search',query,'--account',ACCOUNT,'--no-input','--json']
    if page_token:
        cmd += ['--page', page_token]
    return json.loads(run(cmd))
def gmail_get(msg_id):
    return json.loads(run(['gog','gmail','get',msg_id,'--account',ACCOUNT,'--no-input','--json']))
def decode_b64url(data):
    if not data:
        return ''
    pad = '=' * (-len(data) % 4)
    return base64.urlsafe_b64decode(data + pad).decode('utf-8', errors='ignore')
def extract_parts(payload):
    texts=[]
    htmls=[]
    stack=[payload]
    while stack:
        part=stack.pop()
        mime=part.get('mimeType','')
        body=part.get('body',{})
        data=body.get('data')
        if mime=='text/plain' and data:
            texts.append(decode_b64url(data))
        elif mime=='text/html' and data:
            htmls.append(decode_b64url(data))
        for ch in part.get('parts',[]) or []:
            stack.append(ch)
    return '\n'.join(texts), '\n'.join(htmls)
def strip_html(s):
    s=re.sub(r'<(script|style)[^>]*>.*?</\1>',' ',s, flags=re.I|re.S)
    s=re.sub(r'<br\s*/?>','\n',s, flags=re.I)
    s=re.sub(r'</p>|</div>|</tr>|</li>','\n',s, flags=re.I)
    s=re.sub(r'<[^>]+>',' ',s)
    return html.unescape(s)
def normalize_space(s):
    return re.sub(r'\s+',' ',s).strip()
def parse_headers(headers):
    return {h['name'].lower(): h['value'] for h in headers}
def find_amount(text):
    hits=[]
    for cur,pat in AMOUNT_PATTERNS:
        for m in re.finditer(pat, text, flags=re.I):
            val=float(m.group(1).replace(',',''))
            hits.append((m.start(), cur, val, m.group(0)))
    if not hits:
        return None
    priority=['INR','USD','CAD','GBP','EUR']
    hits.sort(key=lambda x: (priority.index(x[1]) if x[1] in priority else 99, x[0]))
    return hits[0]
def find_context_amount(text, patterns):
    for pat in patterns:
        m=re.search(pat, text, flags=re.I|re.S)
        if not m:
            continue
        groups=[g for g in m.groups() if g is not None]
        if len(groups) == 2:
            cur, raw = groups
        elif len(groups) == 1:
            raw = groups[0]
            cur = 'USD'
        else:
            continue
        try:
            val=float(raw.replace(',',''))
            return (m.start(), cur.upper(), val, m.group(0))
        except:
            continue
    return None
def find_amount_for_class(receipt_class, subject, text, merchant=''):
    combo=' '.join([subject or '', text])
    if merchant == 'Amazon':
        m=re.search(r'\$\s*([0-9][0-9,]*\.?[0-9]{0,2})\s+will be credited to your Amazon account balance', combo, flags=re.I|re.S)
        if m:
            return (m.start(), 'USD', float(m.group(1).replace(',','')), m.group(0))
        m=re.search(r'(?:refund amount|refund total|total refund)\D{0,20}\$\s*([0-9][0-9,]*\.?[0-9]{0,2})', combo, flags=re.I|re.S)
        if m:
            return (m.start(), 'USD', float(m.group(1).replace(',','')), m.group(0))
        totals=[float(x.replace(',','')) for x in re.findall(r'Grand Total:\s*\$\s*([0-9][0-9,]*\.?[0-9]{0,2})', combo, flags=re.I)]
        if totals:
            total=sum(totals)
            return (combo.lower().find('grand total'), 'USD', round(total,2), f"Grand Totals summed: {', '.join(f'{t:.2f}' for t in totals)}")
        return None
    if merchant == 'Airbnb':
        amt = find_context_amount(combo, [
            r'Amount refunded \(USD\) \$\s*([0-9][0-9,]*\.?[0-9]{0,2})',
            r'Total adjustment \(USD\) \$\s*([0-9][0-9,]*\.?[0-9]{0,2})',
            r'Paid \(USD\) \$\s*([0-9][0-9,]*\.?[0-9]{0,2})',
            r'Total \(USD\) \$\s*([0-9][0-9,]*\.?[0-9]{0,2})',
        ])
        if amt:
            return amt
    if merchant in {'Air Canada'}:
        m=re.search(r'Total paid\*?\s*\*?\s*\$\s*([0-9][0-9,]*\.?[0-9]{0,2})\s*(CAD|USD|GBP|EUR|INR)', combo, flags=re.I|re.S)
        if m:
            return (m.start(), m.group(2).upper(), float(m.group(1).replace(',','')), m.group(0))
    if merchant in {'United'}:
        m=re.search(r'Total:\s*([0-9][0-9,]*\.?[0-9]{0,2})\s*(USD|CAD|GBP|EUR|INR)', combo, flags=re.I|re.S)
        if m:
            return (m.start(), m.group(2).upper(), float(m.group(1).replace(',','')), m.group(0))
        m=re.search(r'Total:\s*\$\s*([0-9][0-9,]*\.?[0-9]{0,2})', combo, flags=re.I|re.S)
        if m:
            return (m.start(), 'USD', float(m.group(1).replace(',','')), m.group(0))
    if receipt_class == 'app_store':
        amt = find_context_amount(combo, [
            r'Total:\s*\$\s*([0-9][0-9,]*\.?[0-9]{0,2})(?:/month)?',
            r'Apple Card\s*\$\s*([0-9][0-9,]*\.?[0-9]{0,2})',
        ])
        if amt:
            return amt
    if merchant == 'Budget':
        amt = find_context_amount(combo, [
            r'TOTAL CHARGES\s*\$\s*([0-9][0-9,]*\.?[0-9]{0,2})',
        ])
        if amt:
            return amt
    if merchant == 'Allbirds':
        m=re.search(r'Total \(Paid with Shopify Payments\) \$([0-9][0-9,]*\.?[0-9]{0,2})', combo, flags=re.I|re.S)
        if m:
            return (m.start(), 'USD', float(m.group(1).replace(',','')), m.group(0))
    return find_amount(text)
def convert_to_usd(cur, val):
    if cur=='USD':
        return round(val,2), ''
    rates={'INR':USD_PER_INR,'CAD':USD_PER_CAD,'GBP':USD_PER_GBP,'EUR':USD_PER_EUR}
    usd=round(val*rates[cur],2)
    return usd, f'Original currency: {cur} {val:.2f}'
def clean_merchant(merch):
    merch=normalize_space(merch).strip(' .:-')
    merch=re.sub(r'\*+\d+$','',merch).strip()
    merch=merch.replace('WWW ', '').replace('www.', '')
    merch=merch.replace('.com', '').replace('.in', '')
    merch=normalize_space(merch)
    aliases={
        'swiggy':'Swiggy',
        'swiggy com':'Swiggy',
        'swiggy in':'Swiggy',
        'zomato':'Zomato',
        'chewy':'Chewy',
        'farmina':'Farmina',
        'marshalls pet zone':'Marshalls Pet Zone',
        'amazon':'Amazon',
    }
    low=merch.lower()
    if low in aliases:
        return aliases[low]
    return merch.title()
def extract_forwarded_sender_subject(text):
    sender=''
    subject=''
    m=re.search(r'(?i)from:\s*([^\n]+?)\s+date:', text)
    if m:
        sender=normalize_space(m.group(1))
    m=re.search(r'(?i)subject:\s*([^\n]+?)\s+to:', text)
    if m:
        subject=normalize_space(m.group(1))
    return sender, subject
def classify_receipt(subject, text):
    fwd_sender, fwd_subject = extract_forwarded_sender_subject(text)
    combo=' '.join([subject or '', text[:2500], fwd_sender, fwd_subject]).lower()
    if 'receipt from apple' in combo and 'apple <no_reply@email.apple.com>' in combo:
        return 'app_store'
    if 'google play' in combo:
        return 'app_store'
    if any(k in combo for k in RETAILER_MERCHANTS):
        return 'retailer'
    if any(k in combo for k in TRAVEL_MERCHANTS):
        return 'travel'
    if any(k in combo for k in DIRECT_MERCHANTS):
        return 'direct'
    return 'generic'
def infer_apple_receipt_merchant(subject, text):
    combo=(subject+' '+text).lower()
    if 'receipt from apple' not in combo:
        return None
    lines=[ln.strip() for ln in re.split(r'[\r\n]+', text) if ln.strip()]
    for i, line in enumerate(lines):
        if re.match(r'(?i)apple account\s*:', line):
            for candidate in lines[i+1:i+6]:
                c=normalize_space(re.sub(r'\[image:[^\]]*\]', '', candidate)).strip(' .:-')
                low=c.lower()
                if not c:
                    continue
                if re.search(r'\$\s*[0-9]', c):
                    continue
                if low.startswith('renews '):
                    continue
                if low.startswith('billing and payment'):
                    continue
                if 'apple card' in low:
                    continue
                if '[spender name 1]' in low or '[spender name 2]' in low:
                    continue
                if len(c) > 80:
                    continue
                if low in {'receipt', 'apple'}:
                    continue
                if 'chatgpt' in low:
                    return 'ChatGPT'
                if 'apple one' in low:
                    return 'Apple One'
                return c
    m=re.search(r'(?i)receipt\s+[A-Za-z]+\s+\d{1,2},\s*\d{4}\s+order id:.*?apple account:.*?(?:\[image:[^\]]*\]\s*)?([A-Za-z0-9][A-Za-z0-9+ .&()\-]{2,80}?)\s+(?:renews\s+[A-Za-z]+\s+\d{1,2},\s*\d{4}\s+)?(?:[spender name 1|Spender name 2])\b', text)
    if m:
        c=normalize_space(m.group(1)).strip(' .:-')
        if c:
            if 'chatgpt' in c.lower():
                return 'ChatGPT'
            if 'apple one' in c.lower():
                return 'Apple One'
            return c
    return None
def infer_generic_merchant(subject, text):
    m=re.search(r'towards\s+(.+?)\s+on\s+\d{1,2}\s+[A-Za-z]{3,9},\s*\d{4}\s+at\s+\d{1,2}:\d{2}(?::\d{2})?', text, flags=re.I)
    if m:
        return clean_merchant(m.group(1))
    for pat in MERCHANT_PATTERNS:
        m=re.search(pat, text, flags=re.I)
        if m:
            merch=clean_merchant(m.group(1))
            if merch:
                return merch
    combo=(subject+' '+text).lower()
    if 'swiggy' in combo: return 'Swiggy'
    if 'zomato' in combo: return 'Zomato'
    if 'chewy' in combo: return 'Chewy'
    if 'farmina' in combo: return 'Farmina'
    if 'marshalls pet zone' in combo: return 'Marshalls Pet Zone'
    if 'amazon' in combo: return 'Amazon'
    return None
def infer_merchant(subject, text):
    receipt_class = classify_receipt(subject, text)
    combo=' '.join([subject or '', text[:2500]]).lower()
    if receipt_class == 'app_store':
        apple_merchant = infer_apple_receipt_merchant(subject, text)
        if apple_merchant:
            return apple_merchant
        if 'google one' in combo:
            return 'Google One'
        return 'Google Play'
    if receipt_class == 'retailer':
        for key, merchant in RETAILER_MERCHANTS.items():
            if key in combo:
                return merchant
    if receipt_class == 'travel':
        for key, merchant in TRAVEL_MERCHANTS.items():
            if key in combo:
                return merchant
    if receipt_class == 'direct':
        for key, merchant in DIRECT_MERCHANTS.items():
            if key in combo:
                return merchant
    return infer_generic_merchant(subject, text)
def infer_date_time(headers, text):
    tx=re.search(r'on\s+(\d{1,2}\s+[A-Za-z]{3,9},\s*\d{4})\s+at\s+(\d{1,2}:\d{2}(?::\d{2})?)', text, flags=re.I)
    if tx:
        d=datetime.datetime.strptime(tx.group(1), '%d %b, %Y').date()
        return d, tx.group(2)
    candidates=[]
    for pat in DATE_PATTERNS:
        for m in re.finditer(pat, text):
            candidates.append((m.start(), m.group(1)))
    parsed=[]
    for pos,val in candidates:
        for fmt in ('%d/%m/%Y','%d/%m/%y','%m/%d/%Y','%m/%d/%y','%d-%m-%Y','%d-%m-%y','%d %b %Y','%d %B %Y','%b %d, %Y','%B %d, %Y'):
            try:
                dt=datetime.datetime.strptime(val, fmt)
                if dt.year in (2025, 2026):
                    parsed.append((pos, dt.date(), val))
                break
            except:
                pass
    msg_date=parsed[0][1] if parsed else None
    if not msg_date:
        try:
            msg_date=parsedate_to_datetime(headers.get('date')).date()
        except:
            msg_date=None
    tm=''
    for pat in TIME_PATTERNS:
        m=re.search(pat,text)
        if m:
            tm=m.group(1).upper().replace('  ',' ')
            break
    return msg_date, tm
def looks_like_bad_merchant(merchant):
    m=(merchant or '').strip()
    low=m.lower()
    bad_phrases=[
        'all communications', 'terms of sale', 'privacy policy', 'applecare',
        'lost or stolen', 'claims process', 'distributing or taking any action',
        'manage your', 'subject to credit approval', 'supports and has the latest version',
        'is lost or misused during transmission'
    ]
    if not m:
        return True
    if len(m) > 60:
        return True
    if len(m.split()) > 8:
        return True
    if any(p in low for p in bad_phrases):
        return True
    if m.endswith('.'):
        return True
    if re.search(r'\b(and|or|if|when|while|does|must|should|can|will|with|throughout|during)\b', low) and len(m.split()) > 4:
        return True
    return False
def is_refund_receipt(subject, text, merchant=''):
    combo=' '.join([subject or '', text]).lower()
    refund_markers=[
        'refund', 'amount refunded', 'refunded', 'refund issued', 'advance refund',
        'your refund', 'refund confirmation', 'fully refunded', 'refund processed'
    ]
    if any(m in combo for m in refund_markers):
        return True
    return False
def extract_notes(subject, text, merchant):
    combo=' '.join([subject or '', text[:4000]])
    low=combo.lower()
    if merchant == 'Amazon':
        m=re.search(r'Ordered:\s*"([^"]+)"(?:\s*and\s*(\d+)\s+more items?)?', combo, flags=re.I)
        if m:
            first=m.group(1).strip()
            more=m.group(2)
            return f"Items: {first}" + (f" and {more} more" if more else '')
    if merchant == 'Airbnb':
        loc=re.search(r'Your receipt from Airbnb.*?Receipt ID:.*?\d{4}\s+([^\n]+?)\s+\d+\s+nights?\s+in\s+([^\n]+?)\s', combo, flags=re.I)
        if loc:
            return f"Stay: {normalize_space(loc.group(1))}; Location: {normalize_space(loc.group(2))}"
        m=re.search(r'([A-Za-z][A-Za-z .&\-]+)\s+\d+\s+nights?\s+in\s+([A-Za-z][A-Za-z .&\-]+)', combo)
        if m:
            return f"Stay: {normalize_space(m.group(1))}; Location: {normalize_space(m.group(2))}"
    if merchant == 'Google One':
        return 'Via Google Play'
    if merchant == 'Google Play':
        m=re.search(r'Item Price\s+(.+?)\s+\$[0-9]', combo, flags=re.I)
        if m:
            return f"Item: {normalize_space(m.group(1))}"
    if merchant == 'Budget':
        m=re.search(r'Budget Rental Agreement\s*\*?(\d+)\*?', combo, flags=re.I)
        if m:
            return f"Rental Agreement: {m.group(1)}"
    if merchant == 'Allbirds':
        m=re.search(r'Order Summary.*?\*([^*]+)\*\s*\*\$([0-9.,]+)\*', combo, flags=re.I)
        if m:
            return f"Item: {normalize_space(m.group(1))}"
    if merchant == 'Chugh LLP':
        return 'Payment receipt'
    if merchant == 'Toyota Center':
        m=re.search(r'Thank you for your order for\s+(.+?)\s+To:', combo, flags=re.I)
        if m:
            return f"Event: {normalize_space(m.group(1))}"
    if merchant == 'City of Neptune Beach Parking':
        return 'Parking receipt'
    if merchant == 'Above & Beyond':
        return 'Order receipt'
    return ''
def infer_category(merchant, text):
    s=(merchant+' '+text).lower()
    receipt_class = classify_receipt('', text)
    if is_refund_receipt('', text, merchant):
        return 'Refund', 'Refund'
    merchant_low=(merchant or '').lower()
    if receipt_class == 'retailer':
        if merchant_low in {'amazon','target','walmart','allbirds','lululemon','myntra'}:
            cat='Shopping'
        elif any(k in s for k in ['chewy','farmina','petsmart','pet zone','petzone','marshalls pet zone']):
            cat='Pets'
        else:
            cat='Shopping'
    elif receipt_class == 'travel':
        if any(k in s for k in ['uber','lyft','ola','gas','shell','bp','fuel','parking','toll','fastag']):
            cat='Transportation'
        else:
            cat='Travel'
    elif receipt_class == 'app_store':
        cat='Subscriptions'
    else:
        cat='Shopping'
        for c,keys in CATEGORY_KEYWORDS:
            if any(k in s for k in keys):
                cat=c
                break
    sub=''
    if receipt_class == 'retailer' and cat == 'Shopping' and merchant_low in {'amazon','target','walmart'}:
        sub=''
    else:
        for c,keys in SUBCATEGORY_HINTS:
            if any(k in s for k in keys):
                sub=c
                break
    return cat, sub
def month_week(dateobj):
    month=dateobj.strftime('%B %Y')
    monday=dateobj - datetime.timedelta(days=dateobj.weekday())
    sunday=monday + datetime.timedelta(days=6)
    return month, f"{monday.strftime('%b')} {monday.day}-{sunday.day}"
def load_sheet_rows():
    obj=json.loads(run(['gog','sheets','get',SHEET,'Transactions!A:M','--account',ACCOUNT,'--no-input','--json']))
    return obj.get('values',[])
def canonical_date(s):
    for fmt in ('%Y-%m-%d','%m/%d/%Y','%m/%d/%y'):
        try:
            return datetime.datetime.strptime(s,fmt).date()
        except:
            pass
    return None
def apply_message_label(message_id, label_name):
    run(['gog','gmail','messages','modify',message_id,'--account',ACCOUNT,'--no-input','--add',label_name])
def iso_to_gmail_after(iso_ts):
    try:
        dt=datetime.datetime.fromisoformat(iso_ts.replace('Z','+00:00')).astimezone(datetime.timezone.utc)
        return dt.strftime('%Y/%m/%d')
    except:
        return '2025/12/31'
def parse_search_date(s):
    try:
        return datetime.datetime.strptime(s, '%Y-%m-%d %H:%M').replace(tzinfo=datetime.timezone.utc)
    except:
        return None
def main():
    from_query=BASE_FROM_QUERY
    rows=load_sheet_rows()
    existing=set()
    for r in rows[1:]:
        if len(r)<4:
            continue
        d=canonical_date(r[0])
        merch=(r[2] if len(r)>2 else '').strip().lower()
        amt=(r[3] if len(r)>3 else '').strip()
        try:
            amt=f"{float(amt):.2f}"
        except:
            continue
        if d and merch:
            existing.add((d.isoformat(), merch, amt))
    processed=logged=sk_dup=reviewed=0
    sk_noamt=0
    new_rows=[]
    success_message_ids=[]
    page=None
    seen=0
    while True:
        res=gmail_search_messages(from_query, page)
        messages=res.get('messages') or []
        if not messages:
            break
        for m in messages:
            if seen>=MAX_MESSAGES:
                break
            seen+=1
            msg=gmail_get(m['id'])
            payload=msg.get('payload',{})
            headers=parse_headers(payload.get('headers',[]))
            subject=headers.get('subject','')
            sender=headers.get('from','')
            processed+=1
            if '@amazon' in sender.lower():
                apply_message_label(m['id'], '[Enter Review label name]')
                reviewed+=1
                continue
            full_body=msg.get('body','') or ''
            plain, html_part=extract_parts(payload)
            if re.search(r'(total|grand total|amount paid|amount due|charged|refund amount|payment|paid|debited|rs\.?|\$|£|€|₹)', full_body, flags=re.I):
                text=full_body
            elif re.search(r'(total|grand total|amount paid|amount due|charged|refund amount|payment|paid|debited|rs\.?|\$|£|€|₹)', plain or '', flags=re.I):
                text=plain
            else:
                text=strip_html(html_part)
            text=normalize_space(text)
            merch=infer_merchant(subject, text)
            receipt_class = classify_receipt(subject, text)
            amt=find_amount_for_class(receipt_class, subject, text, merch or '')
            if not amt:
                sk_noamt+=1
                apply_message_label(m['id'], '[Enter Review label name]')
                reviewed+=1
                continue
            _, cur, val, _raw = amt
            if not merch or looks_like_bad_merchant(merch):
                sk_noamt+=1
                apply_message_label(m['id'], '[Enter Review label name]')
                reviewed+=1
                continue
            d, tm=infer_date_time(headers, text)
            if not d:
                sk_noamt+=1
                apply_message_label(m['id'], '[Enter Review label name]')
                reviewed+=1
                continue
            usd, note=convert_to_usd(cur,val)
            refund = is_refund_receipt(subject, text, merch)
            extra_note=extract_notes(subject, text, merch)
            if note and extra_note:
                note = note + '; ' + extra_note
            elif extra_note:
                note = extra_note
            cat, sub=infer_category(merch, text)
            if refund:
                usd = -abs(usd)
                cat = 'Refund'
                if not sub:
                    sub = 'Refund'
            key=(d.isoformat(), merch.lower(), f"{usd:.2f}")
            if key in existing:
                sk_dup+=1
                continue
            month, week=month_week(d)
            row=[d.strftime('%m/%d/%Y'), tm, merch, f"{usd:.2f}", 'USD', cat, sub, '[Spender Name]', '', 'email', note, month, week]
            new_rows.append(row)
            success_message_ids.append(m['id'])
            existing.add(key)
            logged+=1
        if seen>=MAX_MESSAGES:
            break
        page=res.get('nextPageToken')
        if not page:
            break
    append_output=''
    if new_rows:
        values=json.dumps(new_rows, ensure_ascii=False)
        append_output=run(['gog','sheets','append',SHEET,'Transactions!A:M','--account',ACCOUNT,'--no-input','--values-json',values])
        if 'Appended' not in append_output:
            raise RuntimeError('Append did not confirm success:\n'+append_output)
    for mid in success_message_ids:
        apply_message_label(mid, '[Enter Logged label name]')
    print(json.dumps({
        'processed': processed,
        'logged': logged,
        'skipped_duplicates': sk_dup,
        'labeled_review': reviewed,
        'skipped_no_amount': sk_noamt,
        'append_output': append_output.strip(),
        'sample_rows': new_rows[:5],
    }, indent=2))
if __name__ == '__main__':
    main()
your-user@your-machine:~$

