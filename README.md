<a href="https://www.cbsnews.com/news/stocks-soar-and-most-americans-just-dont-care/">[![DOW](https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/dow.jpg)](#)</a>
cbsnews.com

# <a href="https://www.kaggle.com/c/two-sigma-financial-news"> Predicting Stocks Using the News</a>
A quant hedge fund, Two Sigma, is sponsoring a Kaggle competition to see if news headlines can be used to help predict stock price movements. The prediction deliverable is a 1 to -1 confidence-level (Up or Down) for the next 10 days for a ~4000 company subset of U.S Listed Companies.




## Outline

**[DATA EXPLORATION](#data-exploration)**  **[DATA PROCESSING](#data-processing)**  **[MODELING](#image-eda)** **[RESULTS](#results)** **[FUTURE WORK](#future-work)** **[ACKNOWLEDGEMENTS](#acknowledgements)** 


	Tools: [Python, Pandas, Numpy, Keras, Tensorflow, Plotly, SKLearn]

## DATA EXPLORATION

The training data are in two dataframes and no other data is allowed:

	* Market Data (2007 to Present) - Provided by Intrinio, contains financial market information such as opening price, closing price, trading volume, calculated returns, etc.

	* News Data (2007 to Present) - Provided by Reuters, contains information about news articles/alerts published about assets, such as article details, sentiment, and other commentary.


###	Market data
4,072,956 rows and 16 features in the market dataset.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap20.png"></img>


	time(datetime64[ns, UTC]) - the current time (in marketdata, all rows are taken at 22:00 UTC)
	assetCode(object) - a unique id of an asset
	assetName(category) - the name that corresponds to a group of assetCodes. These may be "Unknown" if the corresponding assetCode does not have any rows in the news data.
	universe(float64) - a boolean indicating whether or not the instrument on that day will be included in scoring. This value is not provided outside of the training data time period. The trading universe on a given date is the set of instruments that are avilable for trading (the scoring function will not consider instruments that are not in the trading universe). The trading universe changes daily.
	volume(float64) - trading volume in shares for the day
	close(float64) - the close price for the day (not adjusted for splits or dividends)
	open(float64) - the open price for the day (not adjusted for splits or dividends)
	returnsClosePrevRaw1(float64) - see returns explanation above
	returnsOpenPrevRaw1(float64) - see returns explanation above
	returnsClosePrevMktres1(float64) - see returns explanation above
	returnsOpenPrevMktres1(float64) - see returns explanation above
	returnsClosePrevRaw10(float64) - see returns explanation above
	returnsOpenPrevRaw10(float64) - see returns explanation above
	returnsClosePrevMktres10(float64) - see returns explanation above
	returnsOpenPrevMktres10(float64) - see returns explanation above

These features are mostly floats describing current and past activity on the stock,  with assetCodes and Names for keys.

	The market data contains a variety of returns calculated over different timespans. All of the return features in this set of market data have these properties:

		* Returns are always calculated either open-to-open (from the opening time of one trading day to the open of another) or close-to-close (from the closing time of one trading day to the open of another).
		* Returns are either raw, meaning that the data is not adjusted against any benchmark, or market-residualized (Mktres), meaning that the movement of the market as a whole has been accounted for, leaving only movements inherent to the instrument.



#### Target Variable

		returnsOpenNextMktres10(float64) - 10 day, market-residualized return.

		In this competition, you must predict a signed confidence value, (-1, 1), which is multiplied by the market-adjusted return of a given assetCode over a ten day window.

So the submission is confidence values for a binary target.


Here's the breakdown on the "Up or Down" Movement for all ~10 years of training data. No class imbalance here.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap2.png"></img>

And a visualization of it's movement w.r.t to quantiles. The perturbations in 2008-2009 don't look too great. I'll be using post-2009 data to train my model.


<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap3.png"></img>


#### More EDA - Exploring Close prices


Look Ma, I can make stock price graphs! It's interesting to see how the major stock market events seems to affect all quantiles equally. A low-tide beaches all ships.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap17.png"></img>

Here's a plot of 10 totally random companies. We can see that some companies like New Gold have been around since ~2009, but there are a few companies that are new. We also see some companies disappear from the chart, indicating that they were bought, merged, or went bankrupt.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap15.png"></img>



### News Data


	The news data contains information at both the news article level and asset level 

9,328,750 samples and 35 features.

The article level features include:
* AssetCodes relevant to the article,
* Word Counts in the article
* Sentiment Analysis: Positive, Neutral, Negative

<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap19.png"></img>

<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap18.png"></img>

After taking a peak at the data, I took a look at a wordcloud of the first 100,000 articles to get a feel for the content.

<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/wordcloud.png" width="600px" height="600px"></img>

Yep, looks like financial news alright.

But what about this Sentiment Analysis?

#### A quick foray into the Sentimental analysis via sense2vec
	 "Thomson Reuters has delivered a unique capability in Eikon that takes feeds from both Twitter and StockTwits and weights and analyses sentiment using a proprietary methodology."
	
To get sentiment analysis, they clearly aren't using a bag of words approach. The next best guess is they are using a sense-aware neural net that uses context-embedding vectors.


<a href="https://arxiv.org/pdf/1511.06388.pdf">This research paper</a> by Andrew Trask introduces an approach to disambiguating semantic meaning of words. The algorithm used in Reuter's analysis may be similar to this.

Sense2Vec uses supervised labeling combined with unsupervised clustering to assign sentiment to words given its surrounding sentiments.

<a href="https://arxiv.org/pdf/1511.06388.pdf">
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/code/sense2vec2.png"></img>
</a>

Once labeled words are trained on a corpus, you can get the cosine similarities of other words in the corpus. 

<a href="https://arxiv.org/pdf/1511.06388.pdf">
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/code/sense2vec1.png"></img>
</a>

And if they are labeled with sentiment, you can determine the sentiment similarity as well.

<a href="https://arxiv.org/pdf/1511.06388.pdf">
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/code/sense2v.png"></img>
</a>



## DATA PROCESSING

It's time to clean up some data!

First stop: Bad open/close prices

### Price Outliers
1. Take a look at the months with the biggest price change. January looks interesting.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap14.png"></img>
2. Towers Watson opening at 9998 and then closing at 50. Hmmm seems unrealistic.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap12.png"></img>
3. Yep that's bogus 
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap9.png"></img>
4. Kill them with fire!!

	    def _replace_price_outliers(self, market_train=None):
        """
        Hidden Function to replace outlier/incorrect open and close data

        """

        if market_train is None:
            market_train = self.market_train
            trainprep = True
        market_train['dailychange'] = market_train['close']/market_train['open']
        market_train.loc[market_train['dailychange'] < .33,'open'] = market_train['close']
        market_train.loc[market_train['dailychange'] > 2, 'close'] = market_train['open']
        if trainprep:
            self.market_train = market_train
        else:
            return market_train
5. That's better
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/screencap7.png"></img>



Next I had to merge the news data to the trading days. In order to merge the two datasets into one training set, I decided to use the mean of each company's news data as the aggregation function. 

### Merging news to trading days to minimize information loss

News happens every day, I wrote function to pass to Pandas' map so each news article could be assigned to a trading day.

	
	    def _map_trading_day(self, news_date):
        """
        Hidden function for datafame.map.
        Maps the news_date to its respective trading day.
        The self.tradingdays is stored in memory during the 
        prepare_market() method.
        """
        if news_date in self.tradingdays:
            return news_date
        else:   
            values = self.tradingdays - news_date
            mask = values >= 0
            try:
                return self.tradingdays[mask][0]
            except:
                return 0


With the data cleaned and merged, it's time to model!

## MODELING 

### Whole Data Set Baseline
First here's the baseline ROC for LightGBM (defaultish settings) on the entire holdout set. This  performance is good enough for 50th percentile in the competition.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/lgbm.png"></img>


### PCA Visualization 
Hoping to find clusters to build models around, I PCAd the data down to two principal components. And..... I got a blob. 
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/eda/pca.png"></img>

With no clusters identified, I broke the problem down to a single-company classification with the intention of scaling after I found a solution. 

My goal was to eventually use an LSTM as my final Kaggle submission because: 1. LSTMs are proven to be effective at time-series analysis and 2. Deep Learning networks are so hot right now.

### Single Company Modeling -- AAPL
Since clustering yielded no clear groups of companies, I decided to pick a random company as a baseline. I chose AAPL, since they have a very large market cap and have lots of news.

#### LightGBM AAPL (gradient boosted forest)
	 
Since the training data terminated at 12/31/2016, I decided to use the full-year 2016 as a hold out set.

I fitted a Gradient-Boosted machine to AAPL and plotted the ROC for the hold-out set. Eek, not great.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/lgbmAAPL.png"></img>

But I got some Feature Importances, and they helped me think about what was important to the model. The most important feature was the current day's price change, followed by last day's return. Apparently the most recent returns  are the biggest factor in the next 10 day's return.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/lgbm_feat_import_AAPL.png"></img>


##### Logistic Regression - Get some Coefficients for AAPL
Okay it's a binary problem, so logistic regression should work right?
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/log_roc.png"></img>
Not exactly, but at least I got some coefficients!

Looks like recent news activity for AAPL most influence the model
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/log_coefs.png"></img>

#### LSTM - Because Deep Learning is all the rage
With Gradient Boosting and Logistic Regression out of the way, I decided to build an LSTM. 


Long Short-Term Memory Networks are a very deep and nuanced topic. This is a <a href="http://colah.github.io/posts/2015-08-Understanding-LSTMs/">great article</a> to start the learning. They are a subset of recurrent neural networks with 4 gates in each cell to allow for long-term memory, short-term memory, and the augmentation of both for each window.<a href="http://colah.github.io/posts/2015-08-Understanding-LSTMs/">
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/code/LSTM3-chain.png"></img>
</a>

Here's the model I started with, which was based on <a href="https://github.com/jaungiers/LSTM-Neural-Network-for-Time-Series-Prediction">Jakob Aungier's LSTM model.</a> His model was well suited for simple close price predictions on the DOW, so I added another layer and increased the neuron counts to add complexity for this problem. I kept the dropout layers as an optional parameter in case I saw overfitting.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/code/lstmgraph.png"></img>

Here's the result after training on AAPL, with a 60-day time sequence.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/initial_LSTM_ROC_actual.png"></img>

Okay, not bad right? The next problem was scaling up a model for use with thousands of companies. One-hot encoding out for obvious reasons, so the next option was an embedding layer. Basically a lookup table that would spit out a high-dimensional vector when I pass it a company name as an integer (e.g. AAPL=0). In theory, Embedding layers act a weights for "identity" of an object that you pass it. 


	e.g. AAPL = [ -0.038194, -0.24487, 0.72812, -0.39961, 0.083172, 0.043953, -0.39141, 0.3344, -0.57545, 0.087459, 0.28787, -0.06731, 0.30906, -0.26384, -0.13231, -0.20757, 0.33395, -0.33848, -0.31743, -0.48336, 0.1464, -0.37304, 0.34577, 0.052041, 0.44946, -0.46971, 0.02628, -0.54155, -0.15518, -0.14107, -0.039722, 0.28277, 0.14393, 0.23464, -0.31021, 0.086173, 0.20397, 0.52624, 0.17164, -0.082378, -0.71787]

My hope was the the embedding layer would allow the LSTM to learn the idiosyncracies of each stock without having to build a new model for each stock.


##### Embedded Layer for Assetcodes - A Panacea?
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/embedding/3embed_aapl.png"></img>
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/embedding/3embed_advance.png"></img>
So, yeah terrible results. It looked as if any company had different behavior with respect to the features, it would overwrite what the LSTM had learned from previous company.

##### Embedding Results on Sine Waves

To be sure the embedding layer was or wasn't working, I trained the LSTM on 3 different sine curves with their own embeddings.

Here's the baseline wave and prediction after being trained on itself.

<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/sine_waves/sin_multiply_merge_solotrain.png"></img>

Here's the same sine wave after I trained on 2 subsequent non-similar curves with embedding
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/sine_waves/sin_multiple.png"></img>

Here's one of the other curves. You can see it's trying to regress to the mean with the frequency of the original sine wave.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/sine_waves/sinlong_multiple.png"></img>


#### Short-term is Best-Term

My Fiance' is studying for her next actuarial exam, the FSA, and we've been talking about this project for the last week. She mentioned that the best indicator of a stock's price tomorrow is it's stock price today. I decided to test this theory in the model. Since I want to predict the return for the next 10-days I decided to give it a window of 1/2 of the target. 5-day window.

Here are the results for AAPL and 2 random companies.
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/noembed_aapltrain_5day.png"></img>
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/noembedADVANCE_aapltrain_5day.png"></img>
<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/init_pred/noembedALLSTATE_aapltrain_5day.png"></img>

That went much better than expected. I built out my first Kaggle submission with this model.

## RESULTS
The Kaggle score is basically the mean-variance criterion: The mean return divided by the standard deviation of the returns. A positive score is good, bigger is better.


<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/final_pred/kaggle.png"></img>
Well it's positive. In terms of the leaderboard, it's currently 1261/1607 or top 75%! :)

Here's the ROC of the model for a *random* day in the hold out set. (testing on the entire holdout set wasn't feasible before the presentation)

<img src="https://github.com/joeshull/twosigma_kaggle/blob/master/graphics/final_pred/RocLstmAll.png"></img>



## FUTURE WORK
Going forward, I would like to build a "boosted" version of the LSTM. A model that trains on the companies that give the last model a logloss below -.69 threshold. With the current limitations on the Kaggle competition, building a comprehensive, robust model will be difficult

Gaussian Mixture Modeling would be fun.

## AUTHOR
[auth]: #author 
You can follow me on [twitter](https://twitter.com/joeyshull) or just [email](mailto:joseph.shull@gmail.com) me.

## ACKNOWLEDGMENTS
[acc]: acknowledgments

List of people that I would like to thank:

- Jamie Sloat for her endless support.
- Michael Dyer for being a dope keras architect.
- Frank Burkholder for being a fountain of great ideas when I get stuck.


Copyright © 2018 Joe Shull


