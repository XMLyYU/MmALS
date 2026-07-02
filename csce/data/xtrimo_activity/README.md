## Dataset Structure

The dataset is provided in a tabular format (e.g., CSV) containing information about wild-type and mutant amino acid sequences, their corresponding activity labels, and metadata for model training splits and Multiple Sequence Alignment (MSA) rankings.

### Column Descriptions

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `unique_id` | String | A unique identifier for each sequence record. |
| `wt_aa_seq` | String | The wild-type (WT) amino acid sequence. |
| `mt_aa_seq` | String | The mutant (MT) amino acid sequence. |
| `label` | Float | The mutation activity label (target variable for prediction). |
| `v1_split` | String | Data split assignment for the V1 model (e.g., `train`, `val`, `test`, or left blank). |
| `v2_split` | String | Data split assignment for the V2 model (e.g., `train`, `val`, `test`, or left blank). |
| `msa_v1` | Float | The Multiple Sequence Alignment (MSA) ranking or score used in the V1 model. |
| `msa_v2` | Float | The Multiple Sequence Alignment (MSA) ranking or score used in the V2 model. |

### Example Record

```csv
unique_id,wt_aa_seq,mt_aa_seq,label,v1_split,v2_split,msa_v1,msa_v2
01793d743429f1a8f3edef71f60ea5eb,MDITRFK...,MDITRFK...,1.6045877,train,train,7.0,45.0
