# 🧭 Scikit-learn Complete Learning Roadmap

A structured roadmap to master **scikit-learn (sklearn)** — from beginner to expert — including topics, resources, and practice projects.

---

## 📅 PHASE 1: Foundation Setup (1 week)

### 🎯 Goal
Understand what scikit-learn is and how its workflow fits into ML.

### 🧩 Topics
- Scikit-learn architecture & API design
- Loading datasets (`iris`, `digits`, `wine`, etc.)
- `train_test_split`, `cross_val_score`
- Fit → Transform → Predict pattern
- Basics of NumPy, Pandas, and Matplotlib

### 📘 Resources
- [Scikit-learn User Guide](https://scikit-learn.org/stable/user_guide.html)
- [Corey Schafer – Machine Learning with Scikit-learn (YouTube)](https://www.youtube.com/watch?v=7eh4d6sabA0)
- [FreeCodeCamp – Machine Learning with Python (5 hrs)](https://www.youtube.com/watch?v=7eh4d6sabA0)
- *Introduction to Machine Learning with Python* — Andreas Müller & Sarah Guido

### 🧠 Practice
- Load and visualize the Iris dataset
- Train/test a simple `LogisticRegression` model

---

## ⚙️ PHASE 2: Data Preprocessing & Pipelines (2 weeks)

### 🎯 Goal
Learn to clean, encode, scale, and prepare data efficiently.

### 🧩 Topics
- **Preprocessing**
  - `StandardScaler`, `MinMaxScaler`, `SimpleImputer`, `OneHotEncoder`, `LabelEncoder`
  - `PolynomialFeatures`, `Binarizer`
- **Feature Selection**
  - `SelectKBest`, `RFE`, `VarianceThreshold`
- **Pipelines**
  - `Pipeline`, `make_pipeline`, `ColumnTransformer`
  - Custom Transformers (`FunctionTransformer`)

### 📘 Resources
- [Preprocessing & Feature Engineering Guide](https://scikit-learn.org/stable/modules/preprocessing.html)
- [Pipelines & Composite Estimators](https://scikit-learn.org/stable/modules/compose.html)
- [Data School – Pipelines in Scikit-learn (YouTube)](https://www.youtube.com/watch?v=84gqSbLcBFE)
- *Hands-On Machine Learning with Scikit-Learn, Keras & TensorFlow* (Ch. 2–3)

### 🧠 Practice
- Build a pipeline combining preprocessing + model  
- Compare performance before and after scaling

---

## 🧮 PHASE 3: Supervised Learning (3–4 weeks)

### 🎯 Goal
Learn regression & classification algorithms and their evaluation.

### 🧩 Regression Models
- `LinearRegression`, `Ridge`, `Lasso`, `ElasticNet`
- `SVR`, `DecisionTreeRegressor`, `RandomForestRegressor`
- `KNeighborsRegressor`, `GradientBoostingRegressor`

### 🧩 Classification Models
- `LogisticRegression`, `KNeighborsClassifier`
- `DecisionTreeClassifier`, `RandomForestClassifier`
- `SVC`, `GaussianNB`, `MultinomialNB`
- `AdaBoostClassifier`, `GradientBoostingClassifier`

### 🧩 Evaluation Metrics
- `classification_report`, `confusion_matrix`, `accuracy_score`
- `roc_auc_score`, `precision_recall_curve`
- `mean_squared_error`, `r2_score`
- Cross-validation: `cross_val_score`, `KFold`

### 📘 Resources
- [Supervised Learning Overview](https://scikit-learn.org/stable/supervised_learning.html)
- [StatQuest – Logistic Regression, Decision Trees, Random Forests (YouTube)](https://www.youtube.com/user/joshstarmer)
- [Krish Naik – Scikit-learn Regression & Classification Playlist](https://www.youtube.com/@krishnaik06)
- *Python Machine Learning* by Sebastian Raschka (Ch. 3–6)

### 🧠 Practice
- Compare models on the same dataset  
- Tune hyperparameters with `GridSearchCV` and `RandomizedSearchCV`  
- Plot confusion matrix & ROC curve

---

## 🔍 PHASE 4: Unsupervised Learning (2–3 weeks)

### 🎯 Goal
Understand clustering, dimensionality reduction, and unsupervised metrics.

### 🧩 Topics
- **Clustering**: `KMeans`, `DBSCAN`, `AgglomerativeClustering`, `MeanShift`
- **Dimensionality Reduction**: `PCA`, `KernelPCA`, `NMF`
- **Manifold Learning**: `t-SNE`, `Isomap`
- **Metrics**: Silhouette Score, Davies–Bouldin Index

### 📘 Resources
- [Clustering Guide](https://scikit-learn.org/stable/modules/clustering.html)
- [PCA and Decomposition](https://scikit-learn.org/stable/modules/decomposition.html)
- [StatQuest – PCA and Clustering Explained (YouTube)](https://www.youtube.com/watch?v=FgakZw6K1QQ)
- [Kaggle – Unsupervised Learning Course](https://www.kaggle.com/learn/unsupervised-learning)

### 🧠 Practice
- Cluster customers by spending behavior (`KMeans`)  
- Visualize high-dimensional data with PCA + t-SNE

---

## 🧠 PHASE 5: Model Selection, Validation & Optimization (2 weeks)

### 🎯 Goal
Master hyperparameter tuning and ensemble techniques.

### 🧩 Topics
- `GridSearchCV`, `RandomizedSearchCV`
- Custom scorers with `make_scorer`
- Handling imbalance: `SMOTE`, `class_weight`
- Ensemble Learning:
  - `BaggingClassifier`, `VotingClassifier`, `StackingClassifier`
- Model persistence with `joblib.dump()` / `joblib.load()`

### 📘 Resources
- [Model Evaluation & Tuning Guide](https://scikit-learn.org/stable/model_evaluation.html)
- [Data School – GridSearchCV Tutorial (YouTube)](https://www.youtube.com/watch?v=Gol_qOgRqfA)
- *Hands-On Machine Learning* (Ch. 7–8)

### 🧠 Practice
- Tune RandomForest hyperparameters  
- Build stacking ensemble of 3 classifiers  
- Save & reload models with Joblib

---

## ⚡ PHASE 6: Advanced Topics & Customization (3–4 weeks)

### 🎯 Goal
Learn to extend and customize scikit-learn for production-level use.

### 🧩 Topics
- Custom Estimators (`BaseEstimator`, `TransformerMixin`)
- Pipeline debugging and visualization (`set_output`, `get_feature_names_out`)
- Feature Importance:
  - `PermutationImportance`, `SHAP`, `LIME`
- Model interpretability and explainability
- Integration with XGBoost, LightGBM, TensorFlow

### 📘 Resources
- [Developing Custom Estimators](https://scikit-learn.org/stable/developers/develop.html)
- [Custom Transformers in Scikit-learn (TDS Blog)](https://towardsdatascience.com/custom-transformers-in-scikit-learn-3f7838b4f8b7)
- [SHAP & LIME Explained – StatQuest / Krish Naik (YouTube)](https://www.youtube.com/watch?v=0oz9nsL0K7Y)
- *Interpretable Machine Learning* by Christoph Molnar (Free)

### 🧠 Practice
- Build a custom outlier-removal transformer  
- Use SHAP to interpret model predictions  
- Combine Scikit-learn with XGBoost

---

## 🚀 PHASE 7: Projects & Practice (Ongoing)

### 🎯 Goal
Apply all concepts end-to-end in real-world projects.

### 🧩 Project Ideas
1. 🏠 **House Price Prediction** — Regression + Feature Engineering  
2. 📧 **Spam Classifier** — NLP + Naive Bayes + Pipelines  
3. 🛒 **Customer Segmentation** — KMeans + PCA  
4. 💳 **Credit Card Fraud Detection** — Imbalanced data + ROC analysis  
5. 🌐 **Web App (Flask/Streamlit)** — Deploy Scikit-learn model

### 📘 Resources
- [Kaggle Datasets](https://www.kaggle.com/datasets)
- [Scikit-learn Tutorials by DataCamp](https://www.datacamp.com/tutorial/scikit-learn-tutorial-machine-learning)
- [FreeCodeCamp – End-to-End ML Project](https://www.youtube.com/watch?v=7eh4d6sabA0)

---

## 🏁 Final Outcome

By completing this roadmap, you will:
✅ Understand every major Scikit-learn module  
✅ Build and deploy end-to-end ML pipelines  
✅ Have multiple hands-on ML projects for your portfolio

---

**Author:** *Vinayak Sharma*  
**Version:** 1.0  
