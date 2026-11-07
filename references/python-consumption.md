# Python 消费（路径 B 附属）

```bash
python -m pip install -e "$PDF_FIGS_HOME/python"
```

```python
import pdffigures
from pdffigures.tables import tables_to_dataframe
result = pdffigures.extract("paper.pdf")
# tables[].cells → pd.DataFrame：tables_to_dataframe(t)（逐表调用，
# cells 缺失时返回 None——先判空）
```

注意：`tables_to_dataframe` 只对携带 `cells` 的表（`table-g*` 生长表 /
`mineru-table`）返回 DataFrame；图组管道表（`table-N`）返回 None。
