##########################################################
# RFM ile Müşteri Segmentasyonu
# (Customer Segmentation with RFM)
##########################################################

##########################################################
# 1. İŞ PROBLEMİ
##########################################################

"""
Bir e-ticaret şirketi müşterilerini segmentlere ayırmak ve bu segmentlere göre
pazarlama stratejileri geliştirmek istemektedir.

Amaç:
- En değerli müşterileri belirlemek
- Kaybedilme riski olan müşterileri tespit etmek
- Sadakat stratejileri geliştirmek
- Veri odaklı pazarlama aksiyonları almak

Veri Seti:
https://archive.ics.uci.edu/ml/datasets/Online+Retail+II

Online Retail II veri seti İngiltere merkezli bir online mağazanın
01/12/2009 - 09/12/2011 tarihleri arasındaki satış işlemlerini içermektedir.
"""

##########################################################
# 2. VERİYİ ANLAMA
##########################################################

import datetime as dt
import pandas as pd

pd.set_option('display.max_columns', None)
pd.set_option('display.float_format', lambda x: '%.3f' % x)

# Kaggle ortamına uygun yol kullanımı
df_ = pd.read_excel("../input/online-retail-ii/online_retail_II.xlsx",
                    sheet_name="Year 2009-2010")

df = df_.copy()

df.head()
df.shape
df.isnull().sum()


##########################################################
# 3. VERİ HAZIRLAMA
##########################################################

# Eksik değerleri sil
df.dropna(inplace=True)

# İade faturalarını çıkar (Invoice C ile başlıyorsa iade)
df = df[~df["Invoice"].str.contains("C", na=False)]

# Toplam fiyat değişkeni oluştur
df["TotalPrice"] = df["Quantity"] * df["Price"]

df.describe().T


##########################################################
# 4. RFM METRİKLERİNİN HESAPLANMASI
##########################################################

"""
Recency  : Son alışverişten bu yana geçen gün sayısı
Frequency: Toplam alışveriş sayısı
Monetary : Toplam harcama
"""

today_date = dt.datetime(2010, 12, 11)

rfm = df.groupby("Customer ID").agg({
    "InvoiceDate": lambda date: (today_date - date.max()).days,
    "Invoice": lambda num: num.nunique(),
    "TotalPrice": lambda price: price.sum()
})

rfm.columns = ["recency", "frequency", "monetary"]

rfm = rfm[rfm["monetary"] > 0]

rfm.head()


##########################################################
# 5. RFM SKORLARININ HESAPLANMASI
##########################################################

# Recency skoru (küçük değer yüksek skor)
rfm["recency_score"] = pd.qcut(rfm["recency"], 5, labels=[5,4,3,2,1])

# Frequency skoru
rfm["frequency_score"] = pd.qcut(
    rfm["frequency"].rank(method="first"),
    5,
    labels=[1,2,3,4,5]
)

# Monetary skoru
rfm["monetary_score"] = pd.qcut(
    rfm["monetary"],
    5,
    labels=[1,2,3,4,5]
)

# RFM kombinasyonu (RF bazlı segmentleme)
rfm["RFM_SCORE"] = (
    rfm["recency_score"].astype(str) +
    rfm["frequency_score"].astype(str)
)

rfm.head()


##########################################################
# 6. RFM SEGMENTLERİNİN OLUŞTURULMASI
##########################################################

seg_map = {
    r'[1-2][1-2]': 'hibernating',
    r'[1-2][3-4]': 'at_risk',
    r'[1-2]5': 'cant_lose',
    r'3[1-2]': 'about_to_sleep',
    r'33': 'need_attention',
    r'[3-4][4-5]': 'loyal_customers',
    r'41': 'promising',
    r'51': 'new_customers',
    r'[4-5][2-3]': 'potential_loyalists',
    r'5[4-5]': 'champions'
}

rfm["segment"] = rfm["RFM_SCORE"].replace(seg_map, regex=True)

rfm.groupby("segment").agg({
    "recency": ["mean", "count"],
    "frequency": "mean",
    "monetary": "mean"
}).round(2)


##########################################################
# 7. TÜM SÜRECİN FONKSİYONLAŞTIRILMASI
##########################################################

def create_rfm(dataframe, csv=False):

    dataframe = dataframe.copy()
    dataframe.dropna(inplace=True)
    dataframe = dataframe[~dataframe["Invoice"].str.contains("C", na=False)]
    dataframe["TotalPrice"] = dataframe["Quantity"] * dataframe["Price"]

    today_date = dt.datetime(2011, 12, 11)

    rfm = dataframe.groupby("Customer ID").agg({
        "InvoiceDate": lambda date: (today_date - date.max()).days,
        "Invoice": lambda num: num.nunique(),
        "TotalPrice": lambda price: price.sum()
    })

    rfm.columns = ["recency", "frequency", "monetary"]
    rfm = rfm[rfm["monetary"] > 0]

    rfm["recency_score"] = pd.qcut(rfm["recency"], 5, labels=[5,4,3,2,1])
    rfm["frequency_score"] = pd.qcut(
        rfm["frequency"].rank(method="first"),
        5,
        labels=[1,2,3,4,5]
    )
    rfm["monetary_score"] = pd.qcut(rfm["monetary"], 5, labels=[1,2,3,4,5])

    rfm["RFM_SCORE"] = (
        rfm["recency_score"].astype(str) +
        rfm["frequency_score"].astype(str)
    )

    rfm["segment"] = rfm["RFM_SCORE"].replace(seg_map, regex=True)

    rfm = rfm[["recency","frequency","monetary","segment"]]

    if csv:
        rfm.to_csv("rfm_output.csv")

    return rfm


rfm_final = create_rfm(df, csv=False)

rfm_final.head()
