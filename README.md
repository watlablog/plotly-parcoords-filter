# plotly-parcoords-filter

A minimal notebook demo that **filters a pandas DataFrame from Plotly parallel coordinates (`parcoords`) constraints**.

![リストの表示](https://watlab-blog.com/wp-content/uploads/2026/02/parcoords-plot-list.png)

This repo shows how to:

- Draw a parallel coordinates plot with Plotly
- Use **`go.FigureWidget`** to read each axis **`constraintrange`**
- Convert selected ranges into a **pandas boolean mask**
- Display the matching item names in a **notebook output widget** (`ipywidgets.Output`)

---

## Requirements

This example relies on **Notebook-style execution** (Jupyter Notebook/Lab or VSCode Notebook / Interactive Window).

> ⚠️ Running as a plain `.py` script with `fig.show()` **cannot** capture brush/range interactions back into Python.
> If you need `.py` only, consider a Dash app (not included here).

### Python packages

```bash
pip install pandas plotly ipywidgets anywidget
```

Notes:

- `ipywidgets` provides the output area (and other UI widgets) inside notebooks.
- `anywidget` is required by recent Plotly versions to use `FigureWidget` in notebook environments.

---

## Files

- `okashi_sample.csv` *(your data)*  
- `parcoords_filter_demo.ipynb` *(recommended: the notebook demo)*  
  - If you prefer, you can paste the same code into a VSCode Notebook cell.

---

## CSV format

This demo assumes:

- One **name column** (string): `お菓子名` (configurable)
- The rest are numeric columns used as axes in `parcoords`
- A numeric **color column**: `価格[円]` (configurable)

Example header:

```text
お菓子名,価格[円],カロリー[kcal],塩分[g],糖質[g]
ポテトチップス,160,330,1.2,30
...
```

If your CSV contains non-numeric strings in numeric columns, the demo uses `pd.to_numeric(..., errors="coerce")`.

---

## How to run (VSCode Notebook recommended)

1. Open VSCode
2. Open or create an `.ipynb` notebook
3. Paste the demo code into a cell
4. Run the cell
5. Drag (brush) ranges on the parallel coordinates axes  
   → the list of matching items updates in the output area

---

## Demo code (minimal)

```python
# pip install pandas plotly ipywidgets anywidget
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from IPython.display import display
import ipywidgets as widgets

CSV_PATH = "okashi_sample.csv"
NAME_COL = "お菓子名"
COLOR_COL = "価格[円]"

# CSV読み込み
df = pd.read_csv(CSV_PATH, encoding="shift_jis")

# 平行座標図に使う列を決める（名前列以外を数値軸として扱う）
num_cols = [c for c in df.columns if c != NAME_COL]
df[num_cols] = df[num_cols].apply(pd.to_numeric, errors="coerce")

# 平行座標図をプロット
base_fig = px.parallel_coordinates(
    df,
    dimensions=num_cols,
    color=COLOR_COL,
    color_continuous_scale="jet",
)

# FigureWidget化（Notebook上の対話操作をPythonで受け取るため）
fig = go.FigureWidget(base_fig)

# 選択中候補を表示する出力領域
out = widgets.Output()

def _mask_from_fig():
    """constraintrange を読み取り、条件に合う行を True にする mask を作る。"""
    mask = pd.Series(True, index=df.index)
    for dim, col in zip(fig.data[0].dimensions, num_cols):
        cr = dim.constraintrange
        if cr is None or cr == []:
            continue
        if isinstance(cr[0], (list, tuple)):  # 複数範囲（OR）
            in_any = pd.Series(False, index=df.index)
            for lo, hi in cr:
                in_any |= df[col].between(lo, hi)
            mask &= in_any
        else:  # 単一範囲
            lo, hi = cr
            mask &= df[col].between(lo, hi)
    return mask

def update_output(*_):
    """現在の絞り込み条件に合う候補名を Output に表示する。"""
    mask = _mask_from_fig()
    names = df.loc[mask, NAME_COL].tolist()
    with out:
        out.clear_output()
        print(f"選択中: {len(names)} 件")
        for name in names:
            print(name)

# 絞り込み変更時に候補一覧を更新
for dim in fig.data[0].dimensions:
    dim.on_change(update_output, "constraintrange")

# 初回表示
update_output()
display(fig, out)
```

---

## Troubleshooting

### Nothing updates when brushing
- Ensure you are running in a **notebook environment** (Jupyter / VSCode Notebook).
- Ensure `FigureWidget` is used (not `fig.show()`).
- Try restarting the kernel and re-running the cell.

### `Please install anywidget to use the FigureWidget class`
- Install `anywidget`:
  ```bash
  pip install anywidget
  ```

### Widgets do not render in VSCode
- Make sure the **Jupyter extension** is installed/enabled in VSCode.
- In some environments, `ipywidgets` may require enabling widget support; re-open VSCode or restart the kernel.

---

## License
MIT license.
