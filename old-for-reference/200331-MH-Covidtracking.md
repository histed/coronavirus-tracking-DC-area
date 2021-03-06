---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.2'
      jupytext_version: 1.3.1
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Load covidtracking data and make some plots

```python
%reload_ext autoreload
%autoreload 2

import numpy as np
import matplotlib as mpl
import matplotlib.pyplot as plt
import pytoolsMH as ptMH
import pandas as pd
import seaborn as sns
import os,sys
import scipy.io
import scipy.stats as ss
from pathlib import Path
import statsmodels.api as sm
import statsmodels.formula.api as smf
import requests
import json
import datetime, dateutil.parser

sns.set_style('whitegrid')
sys.path.append('../src')

import warnings
warnings.filterwarnings('ignore')

pd.set_option('display.max_rows', None)

mpl.rc('pdf', fonttype=42) # embed fonts on pdf output 

r_ = np.r_
```

## covidtracking.com data

```python
r = requests.get('https://covidtracking.com/api/states/daily')
data = r.json()
ctDf = pd.DataFrame(data)

# save current data
datestr = datetime.datetime.now().strftime('%y%m%d')
ctDf.to_hdf('./ct-data/covidtracking-data-%s.h5'%datestr, key='ct', complevel=9, complib='zlib')
```

```python
params = pd.DataFrame(index={'DC','MD','VA','NY'}, columns=['fullname'])
params.loc[:,'fullname'] = pd.Series({ 'DC': 'District of Columbia', 'MD': 'Maryland', 'VA': 'Virginia'})
params.loc[:,'labYOff'] = pd.Series({ 'DC': -15, 'MD': +10, 'VA': -10, 'NY':-15})
params.loc[:,'labXOff'] = pd.Series({ 'DC': 0, 'MD': 0, 'VA': +5, 'NY': 0})
params.loc[:,'lw'] = pd.Series({ 'DC': 2, 'MD': 2, 'VA': 2, 'NY': 0.8})

display(params)

```

```python
sns.set_style('darkgrid')
fig, ax = plt.subplots(figsize=r_[1,1]*6, dpi=120)


def plot_state(state='DC', ax=ax):
    desIx = ctDf.state == state
    ys = ctDf.loc[desIx,'positive']
    dtV = pd.to_datetime(ctDf.loc[desIx,'date'], format='%Y%m%d')
    xs = (dtV - dtV.iloc[-1])
    ctDf.loc[desIx,'day0'] = xs
    xs = [x.days for x in xs]


    ph, = ax.plot(xs, ys, marker='.', label=state, lw=2, markersize=9)
    if state in params.index:
        ah = ax.annotate(state, xy=(xs[1],ys.iloc[1]), xycoords='data', 
                    xytext=(params.loc[state,'labXOff'], params.loc[state,'labYOff']),
                    textcoords='offset points',
                    color=ph.get_color(),
                    fontweight='bold', 
                    fontsize=12)
        lw = params.loc[state,'lw']
        ph.set_linewidth(lw)
        if lw < 1:
            ph.set_markersize(3)
            ph.set_color('0.4')
            ah.set_color('0.4')


plot_state('DC')
plot_state('MD')
plot_state('VA')
plot_state('NY')

lp = r_[1,2,5]
yt = np.hstack((lp*10,lp*10**2,lp*10**3,lp*10**4,10**5))
plt.yticks(yt,yt)

ax.set_yscale('log')
ax.set_ylabel('Cases', fontsize=13)
ax.set_xlabel('Days', fontsize=13)
plt.setp(ax.get_xticklabels(), fontsize=9)
plt.setp(ax.get_yticklabels(), fontsize=9)
ax.yaxis.set_major_formatter(mpl.ticker.FuncFormatter(lambda x,pos: '{:,.0f}'.format(x)))
ax.get_xlim()

#ax.set_yticklabels(['%d'%x for x in yt])
ax.set_xlim([0,30])
ax.set_ylim([10, ax.get_ylim()[1]])

# inset
from mpl_toolkits.axes_grid1.inset_locator import inset_axes
axins = inset_axes(ax, width=1.5, height=1.5, bbox_to_anchor=(1.0,0.5,0.3,0.3), bbox_transform=ax.transAxes)
plot_state('DC', ax=axins)


```

## todo

- place element of xlim
- 2nd plot 
- etc

```python
params = pd.DataFrame(index={'DC','MD','VA','NY'}, columns=['fullname'])
params.loc[:,'fullname'] = pd.Series({ 'DC': 'District of Columbia', 'MD': 'Maryland', 'VA': 'Virginia'})
params.loc[:,'labYOff'] = pd.Series({ 'DC': -15, 'MD': +10, 'VA': -10, 'NY':-15})
params.loc[:,'labXOff'] = pd.Series({ 'DC': 0, 'MD': 0, 'VA': +5, 'NY': 0})
params.loc[:,'lw'] = pd.Series({ 'DC': 2, 'MD': 2, 'VA': 2, 'NY': 0.8})
#params.loc[:,'xoff'] = pd.Series({ 'DC': -9, 'MD': -6, 'VA': -6, 'NY': -0.3})
params.loc[:,'xoff'] = pd.Series({ 'DC': 0, 'MD': 0, 'VA': 0, 'NY': -1})

display(params)
```

```python
sns.set_style('darkgrid')
fig, ax = plt.subplots(figsize=r_[1,1]*6, dpi=120)

xlim = r_[0,32]

todayx = 0 #26
params.loc[:,'plot_date'] = np.nan
params['plot_date'] = params['plot_date'].astype('object')
def plot_state(state='DC', ax=ax, is_inset=False):
    global todayx
    desIx = ctDf.state == state
    ys = ctDf.loc[desIx,'positive']
    dtV = pd.to_datetime(ctDf.loc[desIx,'date'], format='%Y%m%d')
    print(f'Latest data for {state}: {dtV.iloc[0]}')
    xs = (dtV - dtV.iloc[-1]) 
    xs = r_[[x.days for x in xs]] + params.loc[state, 'xoff'] #- todayx

    ctDf.loc[desIx,'day0'] = xs
    params.loc[state,'plot_data'] = [{'xs':xs, 'ys':ys}]


    ph, = ax.plot(xs, ys, marker='.', label=state, lw=2, markersize=9)
    if state in params.index:
        if is_inset:
            xytext = r_[7,0]
            xy=(xs[0],ys.iloc[0])
        else:
            xytext = (params.loc[state,'labXOff'], params.loc[state,'labYOff'])
            xy=(xs[1],ys.iloc[1])
            
        ah = ax.annotate(state, 
                         xy=xy, xycoords='data', xytext=xytext, textcoords='offset points',
                         color=ph.get_color(),
                         fontweight='bold', fontsize=12)

        lw = params.loc[state,'lw']
        ph.set_linewidth(lw)
        if lw < 1:
            ph.set_markersize(3)
            ph.set_color('0.4')
            ah.set_color('0.4')
    todayx = np.max((todayx, np.max(xs)))
    params.loc[state, 'color'] = ph.get_color()

plot_state('DC')
plot_state('MD')
plot_state('VA')
plot_state('NY')

def fixups(ax=ax):
    lp = r_[1,2,5]
    yt = np.hstack((lp*10,lp*10**2,lp*10**3,lp*10**4,10**5))
    #ax.set_yticks(yt)
    ax.set_yscale('log')
    plt.setp(ax.get_xticklabels(), fontsize=9)
    plt.setp(ax.get_yticklabels(), fontsize=9)
    ax.yaxis.set_major_locator(mpl.ticker.FixedLocator(yt, nbins=len(yt)+1))
    ax.yaxis.set_major_formatter(mpl.ticker.FuncFormatter(lambda x,pos: '{:,.0f}'.format(x)))

    ax.set_xlim(xlim)
    ax.set_ylim([10, ax.get_ylim()[1]])

fixups(ax)

ylim = ax.get_ylim()
ax.set_ylabel('Cases', fontsize=13)
ax.set_xlabel('Days since first positive test', fontsize=13)

# guide lines
#xs = r_[3,xlim[1]+2]
xs = r_[1,10]
dtL = [2,3,4]
for (iD,dt) in enumerate(dtL):
    ys = 2**(xs/dt)
    y2 = ys*10**3.4/(iD+1)
    ax.plot(xs, y2, '--', lw=0.5, color='0.6')
    ax.annotate('%d days to double'%dt, xy=(7,y2[1]/2), xycoords='data', fontsize=8, color='0.6')



# inset
from mpl_toolkits.axes_grid1.inset_locator import inset_axes
asp = ax.get_aspect()
print(asp)
with sns.axes_style('darkgrid'):
    r0 = 1.4
    axins = inset_axes(ax, width=1.3*r0, height=2.2*r0, bbox_to_anchor=(1.2,0.25,0.3,0.6), bbox_transform=ax.transAxes)
axins.set_facecolor('#EAEAF2')
for st in ['DC', 'MD', 'VA']:
    plot_state(st, ax=axins, is_inset=True)
fixups(ax=axins)

axins.set_xlim(r_[todayx-2.9,todayx+1.0])
axins.set_ylim(r_[250,1900]*2.1)

axins.set_xticks([])
#axins.set_yticks((200,500,1000), [''])#(200,500,1000))
#axins.set_yticks([0])
axins.yaxis.set_visible(False)
axins.set_yticklabels([])
axins.xaxis.set_visible(False)

# put doubling time on inset axes
for st in ['DC', 'MD', 'VA']:
    dD = params.loc[st, 'plot_data']
    for tSt in r_[0,1,2]:
        ns = r_[0:2]+tSt
        x0 = np.mean(dD['xs'][ns])
        y0 = np.mean(dD['ys'].iloc[ns])
        slope0 = np.log10(dD['ys'].iloc[ns[0]])-np.log10(dD['ys'].iloc[ns[1]])
        double_time = np.log10(2)/slope0
        pct_rise = (dD['ys'].iloc[ns[0]]/dD['ys'].iloc[ns[1]] * 100) - 100
        print(pct_rise)
        #tStr = f'{double_time:.0f}'
        tStr = f'{pct_rise:.0f}'
        if st == 'MD' and tSt == 0:
            #tStr = 'Doubling        \ntime (days): ' + tStr 
            tStr = 'Growth             \nper day (%): ' + tStr
        axins.annotate(tStr, xy=(x0,y0), va='bottom', ha='right', color=params.loc[st,'color'],
                      xytext=(-1,2), textcoords='offset points')


xs = r_[1,4]
dtL = [2,3,4]
for (iD,dt) in enumerate(dtL): 
    ys = 2**(xs/dt)
    y2 = ys*10**2.3#/(iD+1)
    #axins.plot(xs+24, y2, '--', lw=0.5, color='0.6')
    #ax.annotate('%d days to ddouble'%dt, xy=(7,y2[1]/2), xycoords='data', fontsize=8, color='0.6')

#ax.axvline(todayx, ls=':', color='0.5', lw=0.25)

ax.annotate('April 4', xy=(30,10), xycoords='data', xytext=(0,-30), textcoords='offset points',
            arrowprops=dict(arrowstyle='->', connectionstyle='arc3', color='0.3'), 
            color='0.3', ha='center')

fig.tight_layout()
fig.savefig('./fig-output/ct-%s.png'%datestr, dpi=300, bbox_inches='tight', bbox_extra_artists=[axins], pad_inches=0.5)
            #bbox_inches=r_[0,0,10,15])#, 
```

```python
np.log10(2)/0.07
```

```python
for a in [2,3,4]:
    pct = (2 ** (1/a)-1) * 100
    print(f'{a} days is {pct:.2g} % ')

```

```python
todayx
```

# Positive test rates in MD, DC, VA

```python
from argparse import Namespace
daylabel = 'Days - last is Apr 3'
fig = plt.figure(figsize=r_[1,0.75]*[2,3]*5, dpi=100)
gs = mpl.gridspec.GridSpec(3,2)

ax = plt.subplot(gs[0,0])
datD = {}
for state in ['DC', 'MD', 'VA']:
    desIx = ctDf.state == state
    stDf = ctDf.loc[desIx,:].copy()
    stDf.set_index('date', inplace=True)

    posV = stDf.loc[:,'positive'][::-1]
    negV = stDf.loc[:,'negative'][::-1]
    pdV = np.diff(posV)
    ndV = np.diff(negV)
    # manual adjustments
    if state == 'MD':  # some errors in testing data
        ndV[22] = np.nan
    if state == 'DC':    
        ndV[26] = ndV[27]/2
        ndV[27] = ndV[27]/2
        #negV.loc[20200401] = negV.loc[20200402]/2
    pctPos = pdV/(pdV+ndV)*100        
    datD[state] = Namespace(posV=posV, negV=negV, pdV=pdV, ndV=ndV, pctPos=pctPos)
    plt.plot(pctPos, '.-', label=state)
ax.set_title('Percent of tests positive, DC area')    
#ax.legend(loc=2)
ax.set_ylabel('Positive tests per day (%)')
ax.set_xlabel(daylabel)
# markup line
maxN = len(datD['DC'].pctPos)
meanNDay = 7
desNs = r_[maxN-meanNDay:maxN]
tV = np.hstack([datD[x].pctPos[desNs] for x in datD.keys()])
tM = np.mean(tV)
plt.plot(desNs, tM+desNs*0, color='k', lw=5, ls='-', alpha=1)
# anno it
ax.annotate('mean\npos test rates,\nlast %d days'%meanNDay, 
            xy=(desNs[2],tM), xycoords='data', xytext=(-50,-120), textcoords='offset points',
            arrowprops=dict(arrowstyle='->', connectionstyle='arc3,rad=0.1', color='k'), 
            color='0.3', ha='center')
ax.annotate('no neg tests\nreported by MD here', 
            xy=(16,100), xycoords='data', xytext=(-10,-30), textcoords='offset points',
            arrowprops=dict(arrowstyle='->', connectionstyle='arc3', color='0.3'), 
            color='0.3', ha='center')

ax = plt.subplot(gs[0,1])
ax2 = plt.subplot(gs[1,1])
ax3 = plt.subplot(gs[2,1])
for state in ['DC', 'MD', 'VA']:
    dd = datD[state]
    display(len(dd.posV))
    pH = ax.plot(dd.pdV, '.-', label=state)        
    pH = ax2.plot(dd.ndV, '.-', label=state)            
    pH = ax3.plot(dd.ndV+dd.pdV, '.-', label=state)
ax.legend()
ax.set_title('Positive results per day')
ax.set_ylabel('Results')

ax2.set_title('Negative results per day')
ax2.set_ylabel('Results')

ax3.set_title('Total results per day')
ax3.set_ylabel('Results')
ax3.set_xlabel(daylabel)

fig.suptitle('Positive test rates have remained relatively constant while total tests grow,\n'
             'suggesting testing criteria are stable', 
             fontsize=16, fontname='Roboto', fontweight='light',
             x=0.05, ha='left', va='top')

ax3.annotate('Data notes:\n'
             '• DC on Apr 1 reported zero neg. cases, and \n  on Apr 2 neg. count doubled, so we adjusted each\n  day to be half the Apr 3 number.',
              xy=(0.05,0.1), xycoords='figure fraction')

doSave = True
if doSave:
    fig.savefig('./fig-output/testing-%s.png'%datestr, 
            dpi=300, bbox_inches='tight', pad_inches=0.5)
            #bbox_inches=r_[0,0,10,15])#, 


```

```python
plt.figure()

for state in ['DC', 'MD', 'VA']:
    desIx = ctDf.state == state
    ys = ctDf.loc[desIx,'positive']
    dtV = pd.to_datetime(ctDf.loc[desIx,'date'], format='%Y%m%d')
    print(f'Latest data for {state}: {dtV.iloc[0]}')
    xs = (dtV - dtV.iloc[-1]) 
    xs = r_[[x.days for x in xs]] + params.loc[state,
```
